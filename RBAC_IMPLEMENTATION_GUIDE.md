# RBAC Implementation Guide

This guide explains how to migrate from the current **group-based** authorization to **Role-Based Access Control (RBAC)** using Django's built-in Permission system.

---

## Current Setup vs RBAC

| Aspect | Current | RBAC |
|--------|---------|------|
| **Authorization** | Hardcoded checks: `user.groups.filter(name="SuperAdmin")` | Permission checks: `user.has_perm("indstaal.view_inquiry")` |
| **Permission logic** | In Python (`utils/permissions.py`) and TS (`usePermissions.ts`) | In database: Group → Permissions mapping |
| **Adding new permission** | Edit code in multiple places | Add permission, assign to group(s) |
| **Managing roles** | Change group membership | Assign/remove permissions from groups |

---

## Implementation Steps

### 1. Define Custom Permissions

Django's default permissions are model-level (`add_estimation`, `change_estimation`, etc.). For application-level actions (e.g. `publish_qrf`, `submit_estimation_to_sales`), define custom permissions.

**Option A: In a dedicated app (recommended)**

Create `indstaal-backend/authorization/` app:

```python
# authorization/models.py
from django.db import models

class Permission(models.Model):
    """Custom application permissions (not Django auth.Permission)."""
    codename = models.CharField(max_length=100, unique=True)
    name = models.CharField(max_length=255)
    module = models.CharField(max_length=50)  # inquiry, qrf, estimation, etc.

    class Meta:
        db_table = "authorization_permission"
```

**Option B: Use Django's ContentType + Permission (simpler)**

Create a "virtual" content type and add custom permissions via migration:

```python
# accounts/migrations/0xxx_add_custom_permissions.py
from django.db import migrations

def add_custom_permissions(apps, schema_editor):
    from django.contrib.auth.models import Permission
    from django.contrib.contenttypes.models import ContentType
    from accounts.models import User  # or any model for content_type

    # Use User's content type as a placeholder for app-level perms
    ct, _ = ContentType.objects.get_or_create(app_label="accounts", model="user")

    custom_perms = [
        ("view_inquiry", "Can view inquiry"),
        ("create_inquiry", "Can create inquiry"),
        ("delete_inquiry", "Can delete inquiry"),
        ("create_qrf", "Can create QRF"),
        ("edit_qrf", "Can edit QRF"),
        ("publish_qrf", "Can publish QRF"),
        ("assign_qrf", "Can assign QRF"),
        ("delete_qrf", "Can delete QRF"),
        ("create_estimation", "Can create estimation"),
        ("submit_estimation_to_admin", "Can submit estimation to admin"),
        ("submit_estimation_to_sales", "Can submit estimation to sales"),
        ("publish_estimation", "Can publish estimation"),
        ("manage_price_config", "Can manage price config"),
        ("edit_price_config_rate", "Can edit price config rate"),
        ("edit_price_config_conversion", "Can edit price config conversion"),
        ("generate_proposal", "Can generate proposal"),
        ("manage_users", "Can manage users"),
        ("invite_users", "Can invite users"),
    ]
    for codename, name in custom_perms:
        Permission.objects.get_or_create(codename=codename, content_type=ct, defaults={"name": name})
```

---

### 2. Assign Permissions to Groups

Create a management command or migration to assign permissions to existing groups:

```python
# accounts/management/commands/setup_rbac.py
from django.core.management.base import BaseCommand
from django.contrib.auth.models import Group, Permission
from django.contrib.contenttypes.models import ContentType

PERMISSION_MAP = {
    "SuperAdmin": [
        "view_inquiry", "create_inquiry", "delete_inquiry",
        "create_qrf", "edit_qrf", "publish_qrf", "assign_qrf", "delete_qrf",
        "create_estimation", "submit_estimation_to_admin", "submit_estimation_to_sales",
        "publish_estimation", "manage_price_config", "edit_price_config_rate",
        "edit_price_config_conversion", "generate_proposal", "manage_users", "invite_users",
    ],
    "Sales": [
        "view_inquiry", "create_inquiry",
        "create_qrf", "edit_qrf", "publish_qrf", "assign_qrf",
        "publish_estimation", "generate_proposal",
    ],
    "Designer": [
        "edit_qrf", "create_estimation", "submit_estimation_to_admin",
    ],
    "Purchase": [
        "manage_price_config", "edit_price_config_rate",
    ],
}

def run(self):
    ct = ContentType.objects.get(app_label="accounts", model="user")
    for group_name, codenames in PERMISSION_MAP.items():
        group, _ = Group.objects.get_or_create(name=group_name)
        perms = Permission.objects.filter(content_type=ct, codename__in=codenames)
        group.permissions.set(perms)
```

---

### 3. Backend: Permission Classes

Replace group checks with permission checks:

```python
# utils/permissions.py (new RBAC version)
from rest_framework.permissions import BasePermission

class HasPermission(BasePermission):
    """Allow if user has the given permission."""
    def __init__(self, permission_codename: str):
        self.permission_codename = permission_codename

    def has_permission(self, request, view):
        u = request.user
        if not (u and u.is_authenticated):
            return False
        if u.is_superuser:
            return True
        # Django permission format: app_label.codename
        return u.has_perm(f"accounts.{self.permission_codename}")

# Usage in views:
# permission_classes = [HasPermission("publish_qrf")]
```

For composite checks (e.g. "SuperAdmin OR Sales"), create a `HasAnyPermission` class or keep a few composite permission classes that internally use `has_perm`.

---

### 4. API: Return Permissions to Frontend

The login response already returns `permissions` from `serializer.get_permissions(user)`. Ensure it returns your **custom** permission codenames, not just Django's default model permissions.

Update `accounts/serializers.py`:

```python
def get_permissions(self, user: User) -> list[str]:
    # Return only our custom app-level permissions
    from django.contrib.contenttypes.models import ContentType
    ct = ContentType.objects.filter(app_label="accounts", model="user").first()
    if not ct:
        return []
    perms = Permission.objects.filter(
        Q(user=user) | Q(group__user=user),
        content_type=ct
    ).values_list("codename", flat=True)
    return sorted(set(perms))
```

Ensure the user payload (from `/user/` or login) includes `permissions: ["create_inquiry", "edit_qrf", ...]`.

---

### 5. Frontend: Use Permissions Instead of Groups

Update `usePermissions.ts` to derive flags from `user.permissions`:

```typescript
// hooks/usePermissions.ts
export function usePermissions() {
  const { user, loading } = useAppSelector((s) => s.auth);
  const permissions = user?.permissions ?? [];

  const has = (perm: string) => permissions.includes(perm);

  return {
    loading,
    isAuthenticated: !!user,
    permissions,  // raw list for RequirePermission
    has,

    // Derived (backward compatible)
    canSeeInquiry: has("view_inquiry"),
    canCreateInquiry: has("create_inquiry"),
    canDeleteInquiry: has("delete_inquiry"),
    canCreateQrf: has("create_qrf"),
    canEditQrf: has("edit_qrf"),
    canPublishQrf: has("publish_qrf"),
    canAssignQrf: has("assign_qrf"),
    canDeleteQrf: has("delete_qrf"),
    canCreateEstimation: has("create_estimation"),
    canSubmitEstimationToAdmin: has("submit_estimation_to_admin"),
    canSubmitEstimationToSales: has("submit_estimation_to_sales"),
    canPublishEstimation: has("publish_estimation"),
    canManagePriceConfig: has("manage_price_config"),
    canEditPriceConfigRate: has("edit_price_config_rate"),
    canEditPriceConfigConversion: has("edit_price_config_conversion"),
    canGenerateProposal: has("generate_proposal"),
    canManageUsers: has("manage_users"),
    canInviteUsers: has("invite_users"),
  };
}
```

Update `RequirePermission` to accept permission codenames:

```tsx
<RequirePermission allow={["create_inquiry", "view_inquiry"]}>
  ...
</RequirePermission>
```

---

### 6. Migration Strategy

1. **Phase 1**: Add custom permissions + assign to groups. Keep existing permission classes. Verify `user.get_all_permissions()` includes new perms.
2. **Phase 2**: Add new `HasPermission` classes alongside old ones. Migrate views one by one.
3. **Phase 3**: Update frontend to use `permissions` from API. Keep `groups` in response for backward compatibility during rollout.
4. **Phase 4**: Remove old group-based permission classes and group-derived logic.

---

### 7. Django Admin for RBAC

SuperAdmin can manage permissions via Django Admin:

- **Groups** → Assign permissions to each group
- **Users** → Assign users to groups (and optionally direct permissions)

No code changes needed to add a new permission: create it, assign to group(s), and use `HasPermission("new_perm")` in the view.

---

### 8. Optional: django-guardian or django-rbac

For **object-level** permissions (e.g. "Sales can edit only QRFs assigned to them"), consider:

- **django-guardian**: Per-object permissions
- **django-rbac**: More flexible role/permission models

For most use cases, Django's built-in Group + Permission is sufficient.

---

## Quick Reference: Permission Codename → Action

| Codename | Description |
|----------|-------------|
| `view_inquiry` | See inquiry list/detail |
| `create_inquiry` | Create inquiry |
| `delete_inquiry` | Delete inquiry |
| `create_qrf` | Create QRF |
| `edit_qrf` | Edit QRF |
| `publish_qrf` | Publish QRF |
| `assign_qrf` | Assign/reassign designer |
| `delete_qrf` | Delete QRF |
| `create_estimation` | Create estimation |
| `submit_estimation_to_admin` | Submit DRAFT → PENDING_DESIGN_ADMIN |
| `submit_estimation_to_sales` | Submit PENDING_DESIGN_ADMIN → PENDING_SALES |
| `publish_estimation` | Publish estimation |
| `manage_price_config` | Access price config |
| `edit_price_config_rate` | Edit rate (Purchase) |
| `edit_price_config_conversion` | Edit conversion (Admin) |
| `generate_proposal` | Generate proposal PDF |
| `manage_users` | User management |
| `invite_users` | Invite new users |

---

## Files to Modify

| File | Changes |
|------|---------|
| `accounts/migrations/` | New migration for custom permissions |
| `accounts/management/commands/` | `setup_rbac.py` to assign perms to groups |
| `utils/permissions.py` | Add `HasPermission`, migrate views |
| `accounts/serializers.py` | `get_permissions()` to return custom perms |
| `indstaal-frontend/hooks/usePermissions.ts` | Use `permissions` instead of `groups` |
| `indstaal-frontend/components/auth/RequirePermission.tsx` | Accept permission codenames |
| Various `*views.py` | Replace `IsAdminUser` etc. with `HasPermission(...)` |
