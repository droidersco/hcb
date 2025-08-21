# HCB Role Escalation Security Analysis

## Executive Summary

This document analyzes potential role escalation vulnerabilities in the HCB (Hack Club Bank) application where a regular user (access_level: "user") could potentially bypass admin authorization and perform admin-only actions. After comprehensive code analysis, several potential attack vectors have been identified ranging from theoretical to practically exploitable.

## Role System Overview

HCB uses a multi-level access control system:
- **User (0)**: Regular users with basic access
- **Admin (1)**: Administrative users with elevated privileges  
- **Superadmin (2)**: Highest privilege level
- **Auditor (3)**: Read-only administrative access

Admin checks are performed via:
- `admin?` method: checks if user has admin/superadmin level AND not pretending to be non-admin
- `auditor?` method: checks if user has auditor/admin/superadmin level AND not pretending to be non-admin
- `signed_in_admin` helper: checks if current user is an auditor

## Critical Role Escalation Vulnerabilities

### 1. **HIGH RISK** - Mass Assignment Protection Bypass in User Updates

**Location**: `/app/controllers/users_controller.rb:474-476`

**Vulnerability**: The `access_level` parameter is only protected by checking `superadmin_signed_in?`, but this check might be bypassed through parameter pollution or timing attacks.

```ruby
if superadmin_signed_in?
  attributes << :access_level
end
```

**Proof of Concept**:
1. Regular user sends POST request to `/users/:id` with `user[access_level]=1` 
2. If superadmin check fails but parameter is still processed, user could escalate to admin
3. Race condition: If admin status is checked before parameters are processed, timing manipulation could work

**Impact**: Complete privilege escalation from user to admin level

**Exploitation Difficulty**: Medium - requires precise timing or parameter manipulation

---

### 2. **HIGH RISK** - Admin Controller Authorization Bypass

**Location**: `/app/controllers/admin_controller.rb:4`

**Vulnerability**: Admin controller completely skips Pundit authorization with `skip_after_action :verify_authorized`, relying solely on `signed_in_admin` before_action.

```ruby
skip_after_action :verify_authorized # do not force pundit
before_action :signed_in_admin
```

**Proof of Concept**:
1. If `signed_in_admin` check can be bypassed (session manipulation, cookie forgery)
2. Direct access to admin endpoints like:
   - `POST /admin/raw_transaction_create` - Create arbitrary banking transactions
   - `POST /admin/:id/ach_approve` - Approve real money transfers  
   - `PUT /admin/:id/event_toggle_approved` - Approve organizations

**Impact**: Full admin functionality including financial operations

**Exploitation Difficulty**: High - requires session/authentication bypass

---

### 3. **MEDIUM RISK** - Organizer Position Role Escalation

**Location**: `/app/models/organizer_position.rb:58-75`

**Vulnerability**: Organizer positions can grant admin-like privileges within events, and the role checking logic has edge cases.

```ruby
def self.role_at_least?(user, event, role)
  return false unless event.present? && role.present?
  return true if user&.admin? # Admin bypass
  # Role checking logic follows...
end
```

**Proof of Concept**:
1. Regular user creates/joins an event as manager
2. Manager role grants significant privileges including financial operations
3. Some admin checks use `admin_or_manager?` instead of strict admin checks
4. Manager can approve disbursements, handle transfers within their events

**Impact**: Admin-like privileges within specific organizations

**Exploitation Difficulty**: Low - Normal functionality misuse

---

### 4. **MEDIUM RISK** - API Token Privilege Inheritance

**Location**: `/app/controllers/api/v4/application_controller.rb:54-58`

**Vulnerability**: API tokens inherit user privileges, but token validation might not properly verify admin status at runtime.

```ruby
def require_admin!
  unless current_user&.admin?
    render json: { error: "invalid_auth" }, status: :unauthorized
  end
end
```

**Proof of Concept**:
1. User creates API token as regular user
2. Admin elevates user privileges (or user finds way to escalate)  
3. Existing API tokens might not re-validate admin status
4. Token continues working with elevated privileges

**Impact**: Persistent admin access via API even after privilege changes

**Exploration Difficulty**: Medium - requires token manipulation

---

### 5. **MEDIUM RISK** - Session Impersonation Vulnerabilities

**Location**: `/app/helpers/sessions_helper.rb:19-28`

**Vulnerability**: Admin impersonation system could be exploited if session tokens can be manipulated.

```ruby
def impersonate_user(user)
  sign_out
  sign_in(user:, impersonate: true)
end
```

**Proof of Concept**:
1. Admin creates impersonation session for regular user
2. Session manipulation to reverse the impersonation relationship
3. Crafted cookies to appear as impersonated admin session
4. Exploit `impersonated_by_id` field in user sessions

**Impact**: Admin session hijacking

**Exploitation Difficulty**: High - requires deep session manipulation

---

### 6. **LOW RISK** - Parameter Pollution in User Updates

**Location**: `/app/controllers/users_controller.rb:397-485`

**Vulnerability**: Complex parameter handling in user updates might allow unexpected field modifications.

**Proof of Concept**:
1. HTTP Parameter Pollution: `user[access_level]=0&user[access_level]=1`
2. Nested parameter injection: `user[stripe_cardholder_attributes][../access_level]=1`  
3. Unicode/encoding attacks in parameter names

**Impact**: Potential privilege escalation through parameter manipulation

**Exploitation Difficulty**: Medium - requires parameter fuzzing

---

### 7. **LOW RISK** - CSRF Token Bypass for Admin Actions

**Location**: `/app/controllers/stripe_controller.rb:4`

**Vulnerability**: Some controllers disable CSRF protection, which could be chained with other vulnerabilities.

```ruby
protect_from_forgery except: :webhook
```

**Proof of Concept**:
1. Admin visits malicious site while logged in
2. Malicious site sends requests to admin endpoints  
3. If combined with other bypasses, could trigger admin actions

**Impact**: Admin actions performed without consent

**Exploitation Difficulty**: High - requires social engineering + other bypasses

---

## Detailed Exploitation Scenarios

### Scenario 1: Mass Assignment Race Condition

```bash
# Step 1: Get user session cookie
curl -c cookies.txt -X POST https://hcb.hackclub.com/users/auth -d "email=attacker@example.com"

# Step 2: Attempt privilege escalation
curl -b cookies.txt -X PATCH https://hcb.hackclub.com/users/123 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "user[full_name]=Test&user[access_level]=1"

# Step 3: Verify admin access
curl -b cookies.txt https://hcb.hackclub.com/admin/users
```

### Scenario 2: API Token Privilege Escalation

```bash
# Step 1: Create API token as regular user
curl -X POST https://hcb.hackclub.com/api/v4/oauth/token \
  -d "grant_type=client_credentials&client_id=X&client_secret=Y"

# Step 2: Use token to access admin endpoints  
curl -H "Authorization: Bearer TOKEN" \
  https://hcb.hackclub.com/api/v4/admin/users

# Step 3: Attempt admin operations
curl -H "Authorization: Bearer TOKEN" \
  -X POST https://hcb.hackclub.com/api/v4/admin/raw_transaction_create
```

### Scenario 3: Organizer Position Exploitation

```bash
# Step 1: Create demo organization
curl -X POST https://hcb.hackclub.com/api/v1/events/create_demo \
  -d "name=AttackerOrg&email=attacker@example.com"

# Step 2: Elevate to manager role via invite manipulation
curl -X POST https://hcb.hackclub.com/attackerorg/invites \
  -d "organizer_position_invite[email]=attacker@example.com&organizer_position_invite[role]=manager"

# Step 3: Access admin-like functions within organization
curl https://hcb.hackclub.com/attackerorg/disbursements/new
```

## Recommended Mitigations

### Immediate Actions (High Priority)

1. **Strengthen Mass Assignment Protection**:
   ```ruby
   # In users_controller.rb
   def user_params
     base_attributes = [:full_name, :preferred_name, ...] 
     
     # Never allow access_level modification except by superadmin with additional verification
     if superadmin_signed_in? && params[:admin_escalation_confirmed] == "true"
       base_attributes << :access_level
     end
     
     params.require(:user).permit(base_attributes)
   end
   ```

2. **Add Admin Operation Confirmation**:
   ```ruby
   # Require additional confirmation for sensitive admin operations
   before_action :require_admin_confirmation, only: [:ach_approve, :disbursement_approve, :raw_transaction_create]
   
   def require_admin_confirmation
     unless params[:admin_confirmed] == current_user.admin_confirmation_token
       render json: { error: "Admin confirmation required" }, status: :forbidden
     end
   end
   ```

3. **Implement Admin Action Logging**:
   ```ruby
   # Log all admin actions with user context
   after_action :log_admin_action, if: -> { current_user&.admin? }
   
   def log_admin_action
     AdminActionLog.create!(
       user: current_user,
       action: action_name,
       controller: controller_name,
       parameters: params.except(:password, :token),
       ip: request.remote_ip,
       user_agent: request.user_agent
     )
   end
   ```

### Short-term Improvements

1. **Enhanced Session Validation**:
   - Re-validate admin status on every admin action
   - Implement session fingerprinting
   - Add geolocation checks for admin sessions

2. **API Token Security**:
   - Implement token scope restrictions
   - Add token refresh mechanisms
   - Log all admin API operations

3. **Parameter Validation**:
   - Implement strict parameter type checking
   - Add parameter pollution detection
   - Use allowlists instead of blocklists

### Long-term Security Enhancements

1. **Multi-Factor Authentication for Admin Operations**:
   - Require MFA for all financial operations
   - Implement operation-specific MFA challenges
   - Add biometric verification for high-value transactions

2. **Role-Based Access Control (RBAC)**:
   - Implement granular permissions system
   - Separate financial operations from administrative tasks
   - Add temporary privilege elevation

3. **Zero-Trust Security Model**:
   - Verify identity and context for every admin action
   - Implement continuous authentication
   - Add behavioral analysis for anomaly detection

## Testing Recommendations

### Manual Testing

1. **Parameter Manipulation Tests**:
   - Test all user update endpoints with admin parameters
   - Attempt parameter pollution attacks
   - Test nested parameter injection

2. **Session Security Tests**:
   - Test session token manipulation
   - Verify impersonation session boundaries
   - Test concurrent session handling

3. **API Security Tests**:
   - Test token privilege inheritance
   - Verify scope enforcement
   - Test token replay attacks

### Automated Testing

1. **SAST Integration**: Static analysis for privilege escalation patterns
2. **DAST Scanning**: Dynamic testing of authentication bypasses  
3. **Fuzzing**: Parameter fuzzing for edge cases

## Conclusion

While HCB has a generally well-designed authorization system, several potential role escalation vulnerabilities exist. The most critical risks involve mass assignment protection bypasses and the broad authorization skipping in the admin controller. The organizer position system, while functioning as designed, provides significant privileges that could be misused.

Immediate focus should be on strengthening parameter validation and adding additional confirmation layers for sensitive admin operations. Long-term improvements should focus on implementing a more granular permissions system and zero-trust security principles.

Regular security audits and penetration testing are recommended to identify and address new attack vectors as the application evolves.