# HCB (Hack Club Bank) API Endpoints and Security Analysis

## Executive Summary

This document provides a comprehensive analysis of all API endpoints in the HCB (Hack Club Bank) application and identifies potential security vulnerabilities. The application is a financial platform that provides banking services for hackathons and other educational organizations.

## API Versions and Architecture

The application uses multiple API versions:
- **V3 API**: Grape-based REST API (unauthenticated, read-only, public data)
- **V4 API**: Rails controllers-based API (authenticated with OAuth2/API tokens)
- **V1 Legacy API**: Limited legacy endpoints
- **Webhook endpoints**: For external service integrations
- **Admin endpoints**: Administrative functionality

---

## V3 API Endpoints (Grape-based, Public/Unauthenticated)

**Base Path**: `/api/v3/`

### General Endpoints
- `GET /api/v3/flavor` - Get flavor text
- `GET /api/v3/git` - Get build commit information

### Organizations
- `GET /api/v3/organizations` - List transparent organizations
- `GET /api/v3/organizations/{id}` - Get single organization

### Organization Sub-resources
- `GET /api/v3/organizations/{id}/transactions` - List organization transactions
- `GET /api/v3/organizations/{id}/card_charges` - List organization card charges
- `GET /api/v3/organizations/{id}/donations` - List organization donations
- `GET /api/v3/organizations/{id}/transfers` - List organization transfers
- `GET /api/v3/organizations/{id}/invoices` - List organization invoices
- `GET /api/v3/organizations/{id}/ach_transfers` - List organization ACH transfers
- `GET /api/v3/organizations/{id}/checks` - List organization checks
- `GET /api/v3/organizations/{id}/cards` - List organization cards

### Individual Resources
- `GET /api/v3/transactions/{id}` - Get single transaction
- `GET /api/v3/card_charges/{id}` - Get single card charge
- `GET /api/v3/donations/{id}` - Get single donation
- `GET /api/v3/transfers/{id}` - Get single transfer
- `GET /api/v3/invoices/{id}` - Get single invoice
- `GET /api/v3/ach_transfers/{id}` - Get single ACH transfer
- `GET /api/v3/checks/{id}` - Get single check
- `GET /api/v3/cards/{id}` - Get single card
- `GET /api/v3/activities` - List recent activities
- `GET /api/v3/activities/{id}` - Get single activity

### Directory API
- `GET /api/v3/directory/organizations` - List directory organizations

---

## V4 API Endpoints (Authenticated)

**Base Path**: `/api/v4/`
**Authentication**: OAuth2 tokens or API tokens with scopes

### User Endpoints
- `GET /api/v4/user` - Get current user (me)
- `GET /api/v4/user/organizations` - List user's organizations
- `GET /api/v4/user/cards` - List user's cards
- `GET /api/v4/user/card_grants` - List user's card grants
- `GET /api/v4/user/invitations` - List user invitations
- `GET /api/v4/user/invitations/{id}` - Get specific invitation
- `POST /api/v4/user/invitations/{id}/accept` - Accept invitation
- `POST /api/v4/user/invitations/{id}/reject` - Reject invitation
- `GET /api/v4/user/transactions/missing_receipt` - Get transactions missing receipts
- `GET /api/v4/user/available_icons` - Get available user icons

### Users (Admin Only)
- `GET /api/v4/users/{id}` - Get user by ID
- `GET /api/v4/users/by_email/{email}` - Get user by email

### Organizations/Events
- `GET /api/v4/organizations/{id}` - Get organization
- `GET /api/v4/organizations/{id}/cards` - List organization cards
- `GET /api/v4/organizations/{id}/card_grants` - List organization card grants
- `POST /api/v4/organizations/{id}/card_grants` - Create card grant
- `GET /api/v4/organizations/{id}/transactions` - List organization transactions
- `GET /api/v4/organizations/{id}/transactions/{id}` - Get specific transaction
- `PUT /api/v4/organizations/{id}/transactions/{id}` - Update transaction
- `GET /api/v4/organizations/{id}/transactions/{id}/receipts` - Get transaction receipts
- `GET /api/v4/organizations/{id}/transactions/{id}/comments` - Get transaction comments
- `POST /api/v4/organizations/{id}/transactions/{id}/comments` - Create transaction comment
- `GET /api/v4/organizations/{id}/transactions/{id}/memo_suggestions` - Get memo suggestions
- `POST /api/v4/organizations/{id}/transfers` - Create transfer/disbursement
- `POST /api/v4/organizations/{id}/donations` - Create donation
- `GET /api/v4/organizations/{id}/followers` - Get organization followers

### Transactions
- `GET /api/v4/transactions/{id}` - Get transaction

### Receipts
- `GET /api/v4/receipts` - List receipts
- `POST /api/v4/receipts` - Create receipt
- `DELETE /api/v4/receipts/{id}` - Delete receipt

### Cards
- `GET /api/v4/cards/{id}` - Get card
- `PUT /api/v4/cards/{id}` - Update card
- `POST /api/v4/cards` - Create card
- `GET /api/v4/cards/{id}/transactions` - Get card transactions
- `GET /api/v4/cards/{id}/ephemeral_keys` - Get ephemeral keys
- `POST /api/v4/cards/{id}/cancel` - Cancel card

### Card Grants
- `GET /api/v4/card_grants/{id}` - Get card grant
- `PUT /api/v4/card_grants/{id}` - Update card grant
- `POST /api/v4/card_grants/{id}/topup` - Top up card grant
- `POST /api/v4/card_grants/{id}/withdraw` - Withdraw from card grant
- `POST /api/v4/card_grants/{id}/cancel` - Cancel card grant

### Stripe Terminal
- `GET /api/v4/stripe_terminal_connection_token` - Get Stripe terminal connection token

---

## V1 Legacy API Endpoints

**Base Path**: `/api/v1/`

- `POST /api/v1/users/find` - Find user by email
- `POST /api/v1/events/create_demo` - Create demo event
- `GET /api/current_user` - Get current user
- `GET /api/flags` - Get feature flags

---

## Webhook Endpoints

- `POST /twilio/webhook` - Twilio webhook
- `POST /stripe/webhook` - Stripe webhook
- `POST /docuseal/webhook` - DocuSeal webhook
- `POST /webhooks/column` - Column webhook

---

## Authentication Mechanisms

### 1. Session-based Authentication
- Standard Rails session cookies for web interface
- CSRF protection enabled for most endpoints
- Session management through `ApplicationController`

### 2. API Token Authentication
- OAuth2-based API tokens stored in `api_tokens` table
- Tokens have scopes and expiration
- Bearer token authentication via HTTP Authorization header
- Encrypted token storage with blind indexing

### 3. HTTP Token Authentication
- Simple token authentication for legacy API endpoints
- Tokens compared using `ActiveSupport::SecurityUtils.secure_compare`

### 4. Multi-factor Authentication
- TOTP (Time-based One-Time Password)
- WebAuthn support
- SMS-based authentication
- Backup codes

---

## Authorization (Pundit Policies)

The application uses Pundit for authorization with policies for:
- API access policies
- User policies
- Event/Organization policies
- Transaction policies
- Card policies
- Administrative policies

---

## Data Format and Request/Response Patterns

### Request Format
- **Content-Type**: `application/json`
- **Authentication**: Bearer token in Authorization header
- **Pagination**: Query parameters for `page`, `per_page`, `limit`
- **Expansion**: `expand` parameter for including related objects
- **Filtering**: Various query parameters for filtering results

### Response Format
```json
{
  "id": "string",
  "type": "resource_type", 
  "attributes": {
    // resource attributes
  },
  "relationships": {
    // related resources
  }
}
```

### Error Response Format
```json
{
  "error": "error_code",
  "message": "Human readable message",
  "messages": ["array", "of", "validation", "errors"]
}
```

---

## Security Vulnerabilities and Concerns

### 🔴 HIGH SEVERITY

#### 1. Information Disclosure in V3 API
**Location**: `/api/v3/` endpoints
**Issue**: V3 API exposes sensitive financial information without authentication
**Risk**: Public access to transaction details, card information, and financial data
**Mitigation**: Only organizations in "Transparency Mode" are exposed, but this should be clearly documented

#### 2. Webhook Endpoint Security
**Location**: `/stripe/webhook`, `/twilio/webhook`, etc.
**Issue**: Webhook endpoints bypass CSRF protection
**Code Reference**: 
```ruby
# app/controllers/stripe_controller.rb
protect_from_forgery except: :webhook
```
**Risk**: Potential webhook flooding or manipulation
**Mitigation**: Signature verification is implemented for Stripe webhooks

#### 3. Admin Endpoint Exposure
**Location**: Various admin endpoints protected by AdminConstraint
**Issue**: Over 100 admin endpoints with high-privilege operations accessible if authentication bypassed
**Risk**: Complete system compromise, financial fraud, data breach

**Specific High-Risk Endpoints:**
```ruby
# Financial Operations (POST/PUT requests)
POST /admin/:id/ach_approve              # Approve bank transfers
POST /admin/:id/ach_reject               # Reject bank transfers  
POST /admin/:id/ach_send_realtime        # Send real-time ACH transfers
POST /admin/:id/disbursement_approve     # Approve disbursements
POST /admin/:id/disbursement_reject      # Reject disbursements
POST /admin/raw_transaction_create       # Create arbitrary transactions
POST /admin/raw_intrafi_transactions_import # Import bulk transactions
PUT  /admin/:id/event_toggle_approved    # Approve/reject organizations
PUT  /admin/:id/event_reject             # Reject organizations

# Sensitive Data Access (GET requests)
GET /admin/users                         # Access all user PII
GET /admin/stripe_cards                  # View all card details  
GET /admin/bank_accounts                 # View all bank account info
GET /admin/raw_transactions              # View all financial transactions
GET /admin/balances                      # View all organization balances
```

**Admin Interface Exposure:**
```ruby
# config/routes.rb lines 11-19
constraints AdminConstraint do
  mount Audits1984::Engine => "/console"     # Audit log interface
  mount Sidekiq::Web => "/sidekiq"           # Job queue management
  mount Flipper::UI.app(Flipper), at: "flipper" # Feature flags
end
constraints AuditorConstraint do  
  mount Blazer::Engine, at: "blazer"         # Database query interface
  mount SchemaEndpoint.instance => "/schema" # Schema access
end
```

**Current Protection Mechanisms:**
```ruby
# lib/admin_constraint.rb lines 7-19
def self.matches?(request)
  cookies = ActionDispatch::Cookies::CookieJar.build(request, request.cookies)
  session_token = cookies.encrypted[:session_token]
  return false unless session_token.present?
  
  potential_session = UserSession.find_by(session_token:)
  if potential_session
    return potential_session.user&.admin?
  end
  false
end

# app/helpers/sessions_helper.rb lines 143-147  
def signed_in_admin
  unless auditor_signed_in?
    redirect_to auth_users_path(require_reload: true), 
                flash: { error: "You'll need to sign in as an admin." }
  end
end

# app/controllers/admin_controller.rb lines 4-5
skip_after_action :verify_authorized # Bypasses Pundit policies
before_action :signed_in_admin
```

**Vulnerability Details:**
1. **Session Hijacking Impact**: Compromised admin session tokens provide immediate access to financial operations
2. **Authorization Bypass**: AdminController skips Pundit authorization, relying solely on role-based checks
3. **Broad Admin Privileges**: Binary admin status grants access to all admin functions without granular permissions
4. **Financial Operation Risk**: Direct access to approve/reject financial transfers without additional verification

### 🟡 MEDIUM SEVERITY

#### 4. CSRF Protection Bypassed
**Location**: Multiple controllers
**Issue**: Several endpoints skip CSRF protection
**Examples**:
```ruby
skip_before_action :verify_authenticity_token, only: [:start_donation, :make_donation, :finish_donation]
```
**Risk**: Cross-site request forgery attacks

#### 5. Overprivileged API Scopes
**Location**: OAuth2 token system
**Issue**: Limited scope granularity may grant excessive permissions
**Risk**: Token compromise could lead to unauthorized access

#### 6. Rate Limiting Coverage
**Issue**: Rate limiting implemented via Rack::Attack but limited coverage for API endpoints
**Current Implementation**: 
- General requests: 1000 per 5 minutes per IP
- Login attempts: 5 per 20 seconds per IP/email
- Donation endpoints: 100 per 20 seconds per IP
**Risk**: API abuse on unprotected endpoints
**Recommendation**: Extend rate limiting to all API endpoints

#### 7. Input Validation
**Location**: Various controllers accepting user input
**Issue**: Some endpoints may lack comprehensive input validation
**Risk**: Injection attacks, data corruption

### 🟢 LOW SEVERITY

#### 8. Error Information Disclosure
**Location**: V3 API error handling
**Code Reference**:
```ruby
# Only in development mode
msg = if Rails.env.development?
        e.message
      else
        "A server error has occurred."
      end
```
**Issue**: Error messages might leak information in development
**Risk**: Information disclosure

#### 9. Debug Endpoints
**Location**: `/my_ip`, flavor text, git commit endpoints
**Issue**: Information disclosure endpoints
**Risk**: Minor information leakage

---

## Security Best Practices Observed

### ✅ Positive Security Features

1. **Strong Authentication**:
   - Multi-factor authentication support
   - WebAuthn implementation
   - Secure token generation and storage

2. **Authorization**:
   - Comprehensive Pundit policy system
   - Role-based access control
   - Resource-level permissions

3. **Cryptography**:
   - Encrypted token storage
   - Secure random token generation
   - Proper signature verification for webhooks

4. **Input Sanitization**:
   - Parameter filtering in controllers
   - ActiveRecord parameter validation

5. **Security Headers and Rate Limiting**:
   - CSRF protection where appropriate
   - JSON-only API responses
   - Rack::Attack for rate limiting and DDoS protection
   - IP safelisting for trusted sources (office, Stripe webhooks)

---

## Recommendations

### Immediate Actions
1. **Audit V3 API exposure** - Ensure only intended data is publicly accessible
2. **Extend rate limiting** to cover all API endpoints
3. **Review CSRF bypass decisions** - Ensure they're necessary and secure
4. **Add comprehensive input validation** to all endpoints

### Short-term Improvements
1. **Implement API monitoring** and alerting
2. **Add request/response logging** for audit trails
3. **Enhance error handling** to prevent information disclosure
4. **Implement API versioning strategy** for breaking changes

### Long-term Enhancements
1. **API security testing** integration in CI/CD
2. **Regular security audits** of endpoint permissions
3. **Implement API usage analytics** and monitoring
4. **Consider API gateway** for centralized security controls

---

## Testing Recommendations

### Security Testing
- **Penetration testing** of all API endpoints
- **Authentication bypass testing**
- **Authorization testing** with different user roles
- **Input fuzzing** and injection testing
- **Rate limiting testing**

### Automated Testing
- **SAST** (Static Application Security Testing) tools
- **DAST** (Dynamic Application Security Testing) tools
- **Dependency vulnerability scanning**
- **Infrastructure security scanning**

---

## Conclusion

The HCB application implements a sophisticated financial platform with multiple API versions serving different purposes. While many security best practices are followed, there are several areas that require attention, particularly around the public V3 API exposure, webhook security, and comprehensive input validation.

The application's use of modern authentication mechanisms (OAuth2, WebAuthn) and authorization patterns (Pundit) demonstrates a mature approach to security, but the complexity of the financial domain requires ongoing security attention and regular audits.

**Risk Assessment**: Medium - The application has good foundational security but requires attention to several medium-severity issues that could lead to data exposure or unauthorized access.