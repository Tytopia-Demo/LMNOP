# Threat Model: Google Auth Library for Ruby

## Overview

### Service Purpose
The Google Auth Library for Ruby (`googleauth`) is Google's officially supported Ruby client library that provides OAuth 2.0 authorization and authentication capabilities for accessing Google APIs. The library implements multiple authentication mechanisms including Application Default Credentials, Service Account authentication, User Credentials (3-Legged OAuth2), and ID Token verification.

### Service Scope
This library serves as a critical security component that:
- Authenticates applications and users to Google services
- Manages OAuth 2.0 token lifecycle (generation, refresh, and storage)
- Verifies ID tokens using JWT validation
- Supports multiple credential types: service accounts, user credentials, compute engine credentials
- Provides secure token storage mechanisms (file-based and Redis-based)
- Handles credential discovery through Application Default Credentials (ADC)

### Key Components
- **Service Account Credentials**: JWT-based authentication for service-to-service communication
- **User Authorization**: 3-Legged OAuth2 flow for user consent
- **ID Token Verification**: Validates Google-issued identity tokens
- **Token Stores**: Persistent storage for access and refresh tokens
- **Credentials Loader**: Auto-discovery of credentials from environment

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         External Systems                             │
└─────────────────────────────────────────────────────────────────────┘
         │                    │                      │
         │ OAuth2 Flow        │ Token Exchange       │ API Requests
         │                    │                      │
         ▼                    ▼                      ▼
┌────────────────────────────────────────────────────────────────────┐
│                   Google Auth Library (Ruby)                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Entry Points:                                                │  │
│  │  - UserAuthorizer.get_authorization_url()                    │  │
│  │  - ServiceAccountCredentials.make_creds()                    │  │
│  │  - IDTokens::Verifier.verify()                               │  │
│  │  - WebUserAuthorizer.handle_auth_callback()                  │  │
│  │  - Google::Auth.get_application_default()                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Core Processing:                                             │  │
│  │  - JWT Generation & Signing (RS256)                          │  │
│  │  - Token Validation & Verification                           │  │
│  │  - OAuth2 Authorization Code Flow                            │  │
│  │  - Credential Discovery & Loading                            │  │
│  │  - Token Refresh Logic                                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Data Storage:                                                │  │
│  │  - FileTokenStore (local filesystem)                         │  │
│  │  - RedisTokenStore (Redis database)                          │  │
│  │  - Environment Variables (credentials)                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
         │                    │                      │
         │ Store Tokens       │ Retrieve Tokens      │ HTTP Requests
         │                    │                      │
         ▼                    ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Downstream Systems                               │
│  - File System (token storage)                                      │
│  - Redis (token storage)                                            │
│  - Google OAuth2 Endpoints (token.googleapis.com)                   │
│  - Google API Endpoints (*.googleapis.com)                          │
└─────────────────────────────────────────────────────────────────────┘
```

## Dependencies

### Direct Dependencies
| Dependency | Version | Purpose | Security Considerations |
|------------|---------|---------|------------------------|
| faraday | >= 0.17.3, < 2.0 | HTTP client for API requests | Network communication, TLS/SSL handling |
| jwt | >= 1.4, < 3.0 | JWT token encoding/decoding | Cryptographic operations, signature validation |
| memoist | ~> 0.16 | Memoization for performance | Caching sensitive data in memory |
| multi_json | ~> 1.11 | JSON parsing | Input validation, deserialization |
| os | >= 0.9, < 2.0 | OS detection | Environment information disclosure |
| signet | ~> 0.14 | OAuth 2.0 client | Core authentication logic |

### System Dependencies
- Ruby runtime (>= 2.4.0)
- OpenSSL library (for RSA key operations)
- Network access to Google OAuth2 endpoints
- File system access (for FileTokenStore)
- Redis server (optional, for RedisTokenStore)

### External Services
- Google OAuth 2.0 Authorization Server (accounts.google.com)
- Google Token Endpoint (oauth2.googleapis.com)
- Google API Discovery Service (for key verification)
- Google Compute Engine Metadata Service (for GCE credentials)

## Entry Points

### 1. Application Default Credentials Discovery
- **Method**: `Google::Auth.get_application_default(scopes)`
- **Input**: Scopes (array/string)
- **Trust Level**: Application
- **Description**: Auto-discovers credentials from environment, file system, or metadata service

### 2. Service Account Credential Creation
- **Method**: `ServiceAccountCredentials.make_creds(json_key_io:, scope:)`
- **Input**: JSON key file IO, scopes
- **Trust Level**: Application
- **Description**: Creates credentials from service account JSON key

### 3. User Authorization URL Generation
- **Method**: `UserAuthorizer.get_authorization_url(options)`
- **Input**: Login hint, state, callback URL
- **Trust Level**: User
- **Description**: Generates OAuth2 authorization URL for user consent

### 4. OAuth2 Callback Handling
- **Method**: `WebUserAuthorizer.handle_auth_callback(request)`
- **Input**: HTTP request with authorization code
- **Trust Level**: User (via OAuth2 flow)
- **Description**: Processes OAuth2 callback and exchanges code for tokens

### 5. ID Token Verification
- **Method**: `IDTokens::Verifier.verify(token)`
- **Input**: JWT token string
- **Trust Level**: External (untrusted)
- **Description**: Verifies and decodes Google-issued ID tokens

### 6. Token Store Operations
- **Methods**: `TokenStore.store()`, `TokenStore.load()`
- **Input**: User ID, token data
- **Trust Level**: Application
- **Description**: Persists and retrieves OAuth2 tokens

### 7. Environment Variable Credentials
- **Variables**: `GOOGLE_PRIVATE_KEY`, `GOOGLE_CLIENT_EMAIL`, `GOOGLE_CLIENT_ID`
- **Input**: Environment configuration
- **Trust Level**: System
- **Description**: Loads credentials from environment variables

## Exit Points

### 1. Google OAuth2 Token Endpoint
- **Destination**: `https://oauth2.googleapis.com/token`
- **Data**: Authorization code, refresh token, client credentials
- **Protocol**: HTTPS POST
- **Purpose**: Exchange authorization code for access tokens

### 2. Google API Endpoints
- **Destination**: `https://*.googleapis.com/*`
- **Data**: Access tokens in Authorization headers
- **Protocol**: HTTPS
- **Purpose**: API requests with authenticated credentials

### 3. File System (Token Storage)
- **Destination**: Local file system paths
- **Data**: OAuth2 tokens (access and refresh tokens)
- **Format**: YAML files
- **Purpose**: Persistent token storage

### 4. Redis (Token Storage)
- **Destination**: Redis server
- **Data**: OAuth2 tokens (access and refresh tokens)
- **Protocol**: Redis protocol
- **Purpose**: Distributed token storage

### 5. Application Headers/Hash
- **Destination**: Calling application code
- **Data**: Authorization header with Bearer token
- **Method**: `apply()` / `apply!()`
- **Purpose**: Inject credentials into HTTP requests

### 6. GCE Metadata Service
- **Destination**: `http://metadata.google.internal/`
- **Data**: Metadata queries
- **Protocol**: HTTP (internal only)
- **Purpose**: Retrieve credentials on Google Compute Engine

### 7. Logging Output
- **Destination**: Application logs, STDERR
- **Data**: Error messages, debugging information
- **Purpose**: Diagnostics and troubleshooting

## Assets

### Critical Assets

#### 1. Private Keys
- **Description**: RSA private keys for service accounts
- **Format**: PEM-encoded RSA keys
- **Source**: JSON key files, environment variables
- **Risk**: Complete account compromise if exposed
- **Protection**: File permissions, secure storage, encryption at rest

#### 2. Access Tokens
- **Description**: OAuth2 access tokens for API authentication
- **Lifetime**: 1 hour (typical)
- **Format**: Bearer tokens
- **Risk**: Unauthorized API access during token lifetime
- **Protection**: Secure storage, transmission over TLS, short expiry

#### 3. Refresh Tokens
- **Description**: Long-lived tokens to obtain new access tokens
- **Lifetime**: Indefinite (until revoked)
- **Format**: Opaque strings
- **Risk**: Long-term unauthorized access
- **Protection**: Secure storage, encryption, access control

#### 4. Client Secrets
- **Description**: OAuth2 client secrets for application authentication
- **Source**: Client credentials JSON
- **Risk**: Application impersonation
- **Protection**: Secure configuration management, no code commits

#### 5. Service Account JSON Keys
- **Description**: Complete service account credentials
- **Contents**: Private key, client email, project ID
- **Risk**: Full service account compromise
- **Protection**: Secure file permissions (600), secure distribution

### Moderate Assets

#### 6. Authorization Codes
- **Description**: Temporary codes exchanged for tokens
- **Lifetime**: 10 minutes (typical)
- **Risk**: Token theft during exchange window
- **Protection**: Single-use enforcement, HTTPS only

#### 7. User Identifiers
- **Description**: User IDs for token storage keys
- **Risk**: Privacy concerns, user enumeration
- **Protection**: Access control on token stores

#### 8. OAuth2 State Parameters
- **Description**: CSRF protection tokens
- **Risk**: CSRF attacks if predictable
- **Protection**: Cryptographically random generation

### Low Assets

#### 9. Public Keys
- **Description**: Public keys for ID token verification
- **Source**: Google's key endpoints
- **Risk**: Minimal (public information)
- **Protection**: Integrity verification

#### 10. Configuration Data
- **Description**: Scopes, callback URLs, metadata
- **Risk**: Information disclosure
- **Protection**: Standard configuration security

## Trust Levels

### Level 0: Untrusted External
- **Description**: Unauthenticated external entities
- **Examples**: Public internet users, untrusted ID tokens
- **Access**: None - must authenticate
- **Controls**: Input validation, rate limiting, HTTPS only

### Level 1: Authenticated User
- **Description**: Users authenticated via OAuth2 flow
- **Examples**: End users who completed 3-Legged OAuth
- **Access**: Operations scoped to their tokens
- **Controls**: Token validation, scope enforcement, expiry checks

### Level 2: Service Account
- **Description**: Authenticated service accounts
- **Examples**: Applications with service account credentials
- **Access**: Operations permitted by service account IAM roles
- **Controls**: Private key protection, JWT validation, scope verification

### Level 3: Application Code
- **Description**: The integrating Ruby application
- **Examples**: Application using googleauth library
- **Access**: Full library API, credential storage access
- **Controls**: Secure coding practices, dependency updates

### Level 4: System Administrator
- **Description**: System-level access to deployment environment
- **Examples**: DevOps engineers, system admins
- **Access**: File system, environment variables, Redis access
- **Controls**: OS-level security, access logging, principle of least privilege

### Level 5: Google Infrastructure
- **Description**: Google's authentication and API infrastructure
- **Examples**: oauth2.googleapis.com, metadata service
- **Access**: Token issuance, validation, API serving
- **Controls**: Mutual TLS, certificate pinning (potential), HTTPS

## STRIDE Threat List

### Spoofing Threats

#### S1: Stolen Service Account Credentials
- **Description**: Attacker obtains service account JSON key file or private key
- **Impact**: Complete impersonation of service account
- **Affected Assets**: Private keys, service account JSON keys
- **Attack Vector**: File system access, insecure storage, code repository leak
- **Likelihood**: Medium
- **Severity**: Critical

#### S2: Access Token Theft
- **Description**: Attacker intercepts or steals valid access tokens
- **Impact**: Temporary unauthorized API access
- **Affected Assets**: Access tokens
- **Attack Vector**: Network interception, memory dumps, log files
- **Likelihood**: Medium
- **Severity**: High

#### S3: OAuth2 Authorization Code Interception
- **Description**: Attacker intercepts authorization code during callback
- **Impact**: Token theft via code exchange
- **Affected Assets**: Authorization codes
- **Attack Vector**: Network MITM, compromised redirect URI
- **Likelihood**: Low
- **Severity**: High

#### S4: Refresh Token Compromise
- **Description**: Long-lived refresh tokens stolen from storage
- **Impact**: Long-term unauthorized access until token revoked
- **Affected Assets**: Refresh tokens
- **Attack Vector**: Token store compromise, backup exposure
- **Likelihood**: Medium
- **Severity**: Critical

### Tampering Threats

#### T1: JWT Token Manipulation
- **Description**: Attacker modifies JWT claims before signature validation
- **Impact**: Elevated privileges, scope expansion
- **Affected Assets**: ID tokens, JWT tokens
- **Attack Vector**: Algorithm confusion, weak signature validation
- **Likelihood**: Low
- **Severity**: Critical

#### T2: Token Store Data Modification
- **Description**: Attacker modifies stored tokens or associations
- **Impact**: Token substitution, session hijacking
- **Affected Assets**: File token store, Redis token store
- **Attack Vector**: File system write access, Redis write access
- **Likelihood**: Low
- **Severity**: High

#### T3: Configuration File Tampering
- **Description**: Modification of client secrets or configuration files
- **Impact**: Credential theft, application misconfiguration
- **Affected Assets**: Client secrets, JSON key files
- **Attack Vector**: File system access, deployment pipeline compromise
- **Likelihood**: Low
- **Severity**: High

#### T4: Dependency Compromise
- **Description**: Malicious or compromised dependencies introduced
- **Impact**: Library behavior modification, credential exfiltration
- **Affected Assets**: All credentials handled by library
- **Attack Vector**: Supply chain attack, dependency confusion
- **Likelihood**: Low
- **Severity**: Critical

### Repudiation Threats

#### R1: Token Usage Tracking
- **Description**: Insufficient logging of credential usage
- **Impact**: Inability to audit unauthorized access
- **Affected Assets**: All tokens and credentials
- **Attack Vector**: Missing audit logs
- **Likelihood**: Medium
- **Severity**: Medium

#### R2: Credential Lifecycle Audit Gap
- **Description**: No audit trail for credential creation/deletion
- **Impact**: Cannot track credential provisioning/revocation
- **Affected Assets**: Service account keys, OAuth2 clients
- **Attack Vector**: Missing audit logs
- **Likelihood**: Medium
- **Severity**: Medium

### Information Disclosure Threats

#### ID1: Credentials in Logs
- **Description**: Sensitive credentials logged to application logs
- **Impact**: Credential exposure via log files
- **Affected Assets**: Private keys, tokens, client secrets
- **Attack Vector**: Debug logging, error messages
- **Likelihood**: Medium
- **Severity**: Critical

#### ID2: Credentials in Error Messages
- **Description**: Exceptions expose credential fragments
- **Impact**: Partial credential disclosure aids attacks
- **Affected Assets**: All credentials
- **Attack Vector**: Verbose error handling
- **Likelihood**: Medium
- **Severity**: High

#### ID3: Token Store Information Leakage
- **Description**: Unauthorized read access to token storage
- **Impact**: Bulk token theft
- **Affected Assets**: Stored tokens in file system or Redis
- **Attack Vector**: Insecure file permissions, Redis misconfiguration
- **Likelihood**: Medium
- **Severity**: Critical

#### ID4: Environment Variable Exposure
- **Description**: Credentials in environment visible to processes
- **Impact**: Credential theft via process inspection
- **Affected Assets**: Private keys, client secrets from environment
- **Attack Vector**: /proc filesystem, process listing
- **Likelihood**: Medium
- **Severity**: High

#### ID5: Memory Dumping
- **Description**: Credentials extracted from application memory
- **Impact**: Credential theft
- **Affected Assets**: All in-memory credentials
- **Attack Vector**: Core dumps, memory debugging tools
- **Likelihood**: Low
- **Severity**: High

#### ID6: Timing Attacks on Token Validation
- **Description**: Timing differences reveal token validity
- **Impact**: Information about valid token structure
- **Affected Assets**: Token validation logic
- **Attack Vector**: Network timing analysis
- **Likelihood**: Low
- **Severity**: Low

### Denial of Service Threats

#### DOS1: Token Storage Exhaustion
- **Description**: Excessive token storage causes resource exhaustion
- **Impact**: Service unavailability, storage filled
- **Affected Assets**: FileTokenStore, RedisTokenStore
- **Attack Vector**: Unbounded token creation
- **Likelihood**: Medium
- **Severity**: Medium

#### DOS2: Rate Limit Exhaustion
- **Description**: Excessive token refresh requests hit rate limits
- **Impact**: Legitimate requests denied
- **Affected Assets**: OAuth2 endpoints
- **Attack Vector**: Aggressive token refresh, poor retry logic
- **Likelihood**: Medium
- **Severity**: Medium

#### DOS3: Cryptographic Resource Exhaustion
- **Description**: Excessive JWT signing/verification operations
- **Impact**: CPU exhaustion, service slowdown
- **Affected Assets**: RSA signing operations
- **Attack Vector**: Malicious token verification requests
- **Likelihood**: Low
- **Severity**: Medium

#### DOS4: Network Timeout Vulnerabilities
- **Description**: Hanging network calls to Google services
- **Impact**: Thread/resource exhaustion
- **Affected Assets**: HTTP client operations
- **Attack Vector**: Network manipulation, slow endpoints
- **Likelihood**: Low
- **Severity**: Medium

### Elevation of Privilege Threats

#### EOP1: Scope Escalation
- **Description**: Attacker gains broader OAuth2 scopes than authorized
- **Impact**: Unauthorized API access beyond intended permissions
- **Affected Assets**: OAuth2 scopes, tokens
- **Attack Vector**: Scope parameter manipulation, confused deputy
- **Likelihood**: Low
- **Severity**: High

#### EOP2: Service Account Impersonation
- **Description**: User credentials used to impersonate service account
- **Impact**: Access to service account resources
- **Affected Assets**: IAM bindings, delegation mechanisms
- **Attack Vector**: Domain-wide delegation misconfiguration
- **Likelihood**: Low
- **Severity**: Critical

#### EOP3: Token Substitution
- **Description**: Attacker replaces their token with victim's token
- **Impact**: Account takeover
- **Affected Assets**: Token store, user-token associations
- **Attack Vector**: Token store write access, session fixation
- **Likelihood**: Low
- **Severity**: Critical

#### EOP4: JWT Algorithm Confusion
- **Description**: Exploiting algorithm header to bypass signature verification
- **Impact**: Forged tokens accepted as valid
- **Affected Assets**: ID token verification
- **Attack Vector**: "none" algorithm, HS256 confusion with RS256
- **Likelihood**: Low
- **Severity**: Critical

## Countermeasures

### Spoofing Countermeasures

#### SC1: Secure Credential Storage
- **Threats Addressed**: S1, S4
- **Implementation**:
  - Store service account keys with 600 permissions (owner read/write only)
  - Use encrypted file systems for sensitive credential storage
  - Implement proper Redis ACLs and authentication
  - Never commit credentials to version control
  - Use secret management systems (e.g., HashiCorp Vault, GCP Secret Manager)
- **Status**: Implemented (file permissions), Recommended (encryption, secret management)

#### SC2: TLS/HTTPS Enforcement
- **Threats Addressed**: S2, S3
- **Implementation**:
  - Enforce HTTPS for all OAuth2 flows and API requests
  - Use Faraday's SSL verification settings
  - Reject insecure redirect URIs (http://)
  - Implement certificate pinning for critical endpoints
- **Status**: Implemented (HTTPS default)

#### SC3: Token Expiry and Rotation
- **Threats Addressed**: S2, S4
- **Implementation**:
  - Respect access token expiry (default 1 hour)
  - Implement automatic token refresh
  - Provide token revocation mechanisms
  - Limit refresh token lifetime where possible
- **Status**: Implemented

#### SC4: Multi-Factor Authentication
- **Threats Addressed**: S1, S4
- **Implementation**:
  - Encourage MFA for user accounts in OAuth2 flows
  - Document MFA best practices
  - Support advanced protection program users
- **Status**: Recommended (application-level)

### Tampering Countermeasures

#### TC1: JWT Signature Validation
- **Threats Addressed**: T1, EOP4
- **Implementation**:
  - Always validate JWT signatures using RS256
  - Reject "none" algorithm tokens
  - Verify algorithm matches expected type (no HS256 for RS256 keys)
  - Use established JWT libraries (jwt gem)
  - Validate all standard claims (iss, aud, exp, iat)
- **Status**: Implemented

#### TC2: Token Store Integrity
- **Threats Addressed**: T2
- **Implementation**:
  - Set restrictive file permissions on token store files (600)
  - Use Redis authentication and ACLs
  - Implement token store checksums/signatures (optional enhancement)
  - Separate read/write permissions where possible
- **Status**: Partially Implemented (file permissions), Recommended (checksums)

#### TC3: Dependency Integrity
- **Threats Addressed**: T4
- **Implementation**:
  - Use Gemfile.lock for dependency pinning
  - Regularly audit dependencies with bundler-audit
  - Monitor security advisories for dependencies
  - Use private gem servers for internal dependencies
  - Verify gem signatures where available
- **Status**: Implemented (Gemfile), Recommended (audit tooling)

#### TC4: Input Validation
- **Threats Addressed**: T1, T3
- **Implementation**:
  - Validate all JSON inputs against expected schema
  - Sanitize file paths for token stores
  - Validate OAuth2 parameters (state, code, redirect_uri)
  - Reject malformed tokens early
- **Status**: Implemented

### Repudiation Countermeasures

#### RC1: Comprehensive Audit Logging
- **Threats Addressed**: R1, R2
- **Implementation**:
  - Log all credential operations (load, refresh, revoke)
  - Include timestamps, user IDs, and operation types
  - Use structured logging for easy parsing
  - Do NOT log sensitive values (tokens, keys)
  - Implement log retention policies
- **Status**: Recommended (application-level)

#### RC2: OAuth2 Token Metadata
- **Threats Addressed**: R1
- **Implementation**:
  - Store token issuance time and source
  - Track token refresh operations
  - Maintain user-token associations
  - Enable token revocation tracking
- **Status**: Partially Implemented

### Information Disclosure Countermeasures

#### IDC1: Secure Logging Practices
- **Threats Addressed**: ID1, ID2
- **Implementation**:
  - Never log tokens, keys, or client secrets
  - Sanitize error messages to exclude credentials
  - Use log levels appropriately (no debug in production)
  - Implement log scrubbing for sensitive patterns
  - Review all logger calls for credential exposure
- **Status**: Implemented (basic), Recommended (log scrubbing)

#### IDC2: Token Store Access Control
- **Threats Addressed**: ID3
- **Implementation**:
  - File token store: 600 permissions, owner-only access
  - Redis token store: authentication required, ACLs configured
  - Separate token stores per environment
  - Regular access audits
- **Status**: Implemented (file permissions), Recommended (Redis ACLs)

#### IDC3: Environment Variable Security
- **Threats Addressed**: ID4
- **Implementation**:
  - Minimize credential storage in environment variables
  - Use secret management systems instead
  - Restrict process visibility (containers, namespaces)
  - Clear sensitive environment variables after use
- **Status**: Recommended (migration to secrets management)

#### IDC4: Memory Protection
- **Threats Addressed**: ID5
- **Implementation**:
  - Disable core dumps in production
  - Clear sensitive variables after use (where possible)
  - Use memory-safe Ruby practices
  - Implement process isolation
- **Status**: Recommended (system-level)

#### IDC5: Constant-Time Comparisons
- **Threats Addressed**: ID6
- **Implementation**:
  - Use constant-time string comparison for tokens
  - Avoid timing-dependent validation logic
  - Implement rate limiting for validation endpoints
- **Status**: Recommended

### Denial of Service Countermeasures

#### DOSC1: Token Storage Limits
- **Threats Addressed**: DOS1
- **Implementation**:
  - Implement maximum token count per user
  - Automatic cleanup of expired tokens
  - Storage quota monitoring and alerting
  - Implement token store size limits
- **Status**: Recommended

#### DOSC2: Rate Limiting and Backoff
- **Threats Addressed**: DOS2
- **Implementation**:
  - Implement exponential backoff for token refresh
  - Respect Google API rate limits
  - Cache tokens appropriately (memoist usage)
  - Implement circuit breakers for external calls
- **Status**: Partially Implemented (caching)

#### DOSC3: Resource Limits
- **Threats Addressed**: DOS3, DOS4
- **Implementation**:
  - Set HTTP client timeouts (connection, read)
  - Limit concurrent token operations
  - Implement request queuing
  - Monitor CPU usage for cryptographic operations
- **Status**: Recommended

#### DOSC4: Input Size Validation
- **Threats Addressed**: DOS3
- **Implementation**:
  - Limit maximum token size
  - Validate JSON payload sizes
  - Reject oversized inputs early
- **Status**: Recommended

### Elevation of Privilege Countermeasures

#### EOPC1: Strict Scope Validation
- **Threats Addressed**: EOP1
- **Implementation**:
  - Validate requested scopes against allowed scopes
  - Implement principle of least privilege
  - Reject scope expansion in token refresh
  - Document required scopes per operation
- **Status**: Implemented

#### EOPC2: Domain-Wide Delegation Controls
- **Threats Addressed**: EOP2
- **Implementation**:
  - Document domain-wide delegation risks
  - Require explicit configuration for delegation
  - Audit service accounts with delegation permissions
  - Implement just-in-time privilege escalation
- **Status**: Recommended (documentation)

#### EOPC3: Token-User Binding Validation
- **Threats Addressed**: EOP3
- **Implementation**:
  - Verify user ID matches token ownership
  - Validate token subject claims
  - Prevent token sharing across users
  - Implement token binding to client
- **Status**: Implemented

#### EOPC4: Algorithm Whitelist
- **Threats Addressed**: EOP4
- **Implementation**:
  - Only accept RS256 for Google ID tokens
  - Reject "none" algorithm explicitly
  - Validate algorithm in JWT header matches expected
  - Use jwt gem's algorithm verification features
- **Status**: Implemented

### Additional Security Controls

#### ASC1: Security Headers
- **Implementation**:
  - Set secure cookie flags for web applications
  - Implement HSTS for web endpoints
  - Use SameSite cookie attributes
- **Status**: Recommended (application-level)

#### ASC2: Regular Security Audits
- **Implementation**:
  - Conduct periodic security reviews
  - Perform dependency vulnerability scanning
  - Review access logs for anomalies
  - Penetration testing of OAuth2 flows
- **Status**: Recommended

#### ASC3: Incident Response
- **Implementation**:
  - Maintain credential revocation procedures
  - Document incident response playbooks
  - Implement automated token revocation
  - Set up security monitoring and alerting
- **Status**: Recommended

#### ASC4: Developer Training
- **Implementation**:
  - Document secure usage patterns
  - Provide example code with security best practices
  - Highlight common pitfalls
  - Regular security training for maintainers
- **Status**: Implemented (documentation)

## Security Recommendations

### For Library Maintainers
1. Implement automated security scanning in CI/CD pipeline
2. Maintain security advisory mailing list
3. Conduct regular dependency audits
4. Provide security-focused documentation
5. Implement security unit tests for threat scenarios

### For Library Users
1. Never commit credentials to version control
2. Use secret management systems instead of environment variables
3. Implement proper file permissions for token stores (600)
4. Enable audit logging for all credential operations
5. Regularly rotate service account keys
6. Implement least-privilege IAM policies
7. Use short-lived credentials where possible
8. Monitor for credential compromise indicators
9. Keep library and dependencies updated
10. Follow OAuth2 security best practices (PKCE, state parameter)

### Compliance Considerations
- **GDPR**: User tokens may contain personal data; implement retention policies
- **SOC2**: Audit logging and access controls required
- **PCI DSS**: Protect tokens as sensitive authentication data
- **HIPAA**: Encrypt tokens at rest and in transit
- **FedRAMP**: Follow federal security requirements for cloud services

## Review and Maintenance

- **Document Owner**: Security Team
- **Last Review Date**: 2024
- **Review Frequency**: Quarterly
- **Next Review Date**: Q2 2024
- **Version**: 1.0

This threat model should be reviewed and updated whenever:
- New authentication methods are added
- Dependencies are updated or changed
- Security vulnerabilities are discovered
- Architecture changes are made
- New entry/exit points are added
- Compliance requirements change
