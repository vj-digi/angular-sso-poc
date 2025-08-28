# Technical Architecture - VTest OIDC Client SSO POC

## System Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Angular App   │    │   AWS Cognito    │    │  External IdP   │
│  (Frontend)     │◄──►│   (OIDC Server)  │◄──►│  (MS AD)        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Browser       │    │   User Pool      │    │   Digivalet     │
│   Storage       │    │   Management     │    │   OIDC Bridge   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## Detailed Component Analysis

### 1. Authentication Service (`auth.service.ts`)

#### Core Responsibilities
- OIDC client configuration and management
- User authentication flow orchestration
- Token management and validation
- Custom attribute updates via AWS SDK

#### Key Dependencies
```typescript
import { UserManager, User } from 'oidc-client';
import {
  CognitoIdentityProviderClient,
  UpdateUserAttributesCommand
} from '@aws-sdk/client-cognito-identity-provider';
```

#### Configuration Object Structure
```typescript
interface OIDCConfig {
  authority: string;           // Cognito User Pool endpoint
  client_id: string;          // OAuth 2.0 client identifier
  client_secret: string;      // OAuth 2.0 client secret
  redirect_uri: string;       // Callback URL after auth
  response_type: string;      // Authorization code flow
  scope: string;              // Requested permissions
  post_logout_redirect_uri: string; // Post-logout redirect
  extraQueryParams: {         // Additional parameters
    identity_provider: string; // External IdP identifier
  };
}
```

#### Authentication Flow Methods

##### Login Initiation
```typescript
login() {
  return this.userManager.signinRedirect();
}
```
- Redirects user to Cognito hosted UI
- Initiates OIDC authorization code flow
- User is redirected to external IdP for authentication

##### Callback Handling
```typescript
async handleCallback() {
  const user: User = await this.userManager.signinRedirectCallback();
  
  if (user?.access_token) {
    await this.updateCustomAttributes(user.access_token);
  }
  
  return user;
}
```
- Processes authorization code from Cognito
- Exchanges code for access, ID, and refresh tokens
- Automatically updates custom attributes
- Returns user object with tokens

##### Custom Attribute Updates
```typescript
private async updateCustomAttributes(accessToken: string) {
  try {
    const command = new UpdateUserAttributesCommand({
      AccessToken: accessToken,
      UserAttributes: [
        { Name: 'custom:platformUserId', Value: '1234567890' },
        { Name: 'custom:userIdentityId', Value: '1234567890' }
      ]
    });

    const result = await this.cognitoClient.send(command);
    console.log('Custom attributes updated:', result);
  } catch (err) {
    console.error('Error updating custom attributes:', err);
  }
}
```

**Key Points:**
- Uses AWS SDK v3 for Cognito operations
- Requires valid access token for attribute updates
- Custom attributes must follow Cognito naming convention (`custom:attributeName`)
- Error handling for failed attribute updates

### 2. Callback Component (`callback.component.ts`)

#### Responsibilities
- Handles OIDC callback URL routing
- Processes authentication response
- Displays authentication results
- Provides debugging information

#### Lifecycle Flow
```typescript
ngOnInit() {
  // 1. Extract query parameters
  this.route.queryParams.subscribe(params => {
    const code = params['code'];    // Authorization code
    const state = params['state'];  // CSRF protection token
  });

  // 2. Process authentication callback
  this.authService.handleCallback().then(() => {
    // 3. Retrieve user information
    this.authService.getUser().then(user => {
      // 4. Display user details and tokens
      console.log('User object:', user);
      console.log('Access token:', user.access_token);
      console.log('ID token:', user.id_token);
      console.log('User profile:', user.profile);
    });
  });
}
```

#### Query Parameter Processing
- **Code**: Authorization code from Cognito (single-use)
- **State**: CSRF protection token (validates request authenticity)
- **Error**: Error parameter if authentication failed
- **Error Description**: Human-readable error message

### 3. Application Routing (`app.routes.ts`)

#### Route Configuration
```typescript
export const routes: Routes = [
  { path: 'auth/callback', component: CallbackComponent },
];
```

#### OIDC Callback Route
- **Path**: `/auth/callback`
- **Component**: `CallbackComponent`
- **Purpose**: Handle post-authentication redirects
- **Security**: Must match exactly with Cognito configuration

### 4. Main Application Component (`app.component.ts`)

#### Authentication Controls
```typescript
export class AppComponent {
  constructor(private authService: AuthService) {}

  login() {
    this.authService.login();
  }

  logout() {
    this.authService.logout();
  }
}
```

#### User Interface Elements
- Login button: Initiates OIDC flow
- Logout button: Terminates user session
- Router outlet: Displays callback component

## OIDC Flow Implementation

### 1. Authorization Code Flow

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│   User      │    │   Angular    │    │   Cognito   │    │   External   │
│   Browser   │    │   App        │    │   User Pool │    │   IdP        │
└─────────────┘    └──────────────┘    └─────────────┘    └──────────────┘
       │                   │                   │                   │
       │ 1. Click Login    │                   │                   │
       │─────────────────►│                   │                   │
       │                   │ 2. signinRedirect │                   │
       │                   │─────────────────►│                   │
       │                   │                   │ 3. Redirect to    │
       │                   │                   │   External IdP    │
       │                   │                   │─────────────────►│
       │                   │                   │                   │ 4. User Auth
       │                   │                   │                   │◄─────────────
       │                   │                   │ 5. Auth Response  │
       │                   │                   │◄─────────────────│
       │                   │ 6. Redirect with  │                   │
       │                   │    Auth Code      │                   │
       │ 7. Callback URL   │                   │                   │
       │◄─────────────────│                   │                   │
       │                   │ 8. handleCallback │                   │
       │                   │─────────────────►│                   │
       │                   │ 9. Token Exchange │                   │
       │                   │◄─────────────────│                   │
       │                   │ 10. Update Attrs  │                   │
       │                   │─────────────────►│                   │
       │                   │ 11. Attr Response │                   │
       │                   │◄─────────────────│                   │
```

### 2. Token Exchange Process

#### Authorization Code to Token Exchange
```typescript
// Internal to oidc-client library
const tokenResponse = await this.userManager.signinRedirectCallback();
```

**What Happens:**
1. Authorization code is extracted from callback URL
2. POST request to Cognito token endpoint
3. Cognito validates authorization code
4. Returns access, ID, and refresh tokens
5. Tokens are stored securely by oidc-client

#### Token Types and Usage

**Access Token (JWT)**
- **Purpose**: API authentication and authorization
- **Usage**: AWS SDK calls, custom attribute updates
- **Lifetime**: Typically 1 hour
- **Claims**: User permissions, scope information

**ID Token (JWT)**
- **Purpose**: User identity verification
- **Usage**: User profile information, authentication status
- **Lifetime**: Same as access token
- **Claims**: User attributes, authentication time, issuer

**Refresh Token**
- **Purpose**: Token renewal without re-authentication
- **Usage**: Automatic token refresh by oidc-client
- **Lifetime**: Longer than access token (days/weeks)
- **Storage**: Secure browser storage

### 3. Custom Attribute Update Process

#### AWS SDK Integration
```typescript
// Initialize Cognito client
this.cognitoClient = new CognitoIdentityProviderClient({
  region: 'ap-south-1'
});

// Update user attributes
const command = new UpdateUserAttributesCommand({
  AccessToken: accessToken,
  UserAttributes: [
    { Name: 'custom:platformUserId', Value: '1234567890' },
    { Name: 'custom:userIdentityId', Value: '1234567890' }
  ]
});

const result = await this.cognitoClient.send(command);
```

#### Attribute Update Flow
1. **Validation**: Access token is validated by AWS
2. **Authorization**: Token must have required permissions
3. **Processing**: Cognito updates user attributes
4. **Response**: Success/failure confirmation
5. **Error Handling**: Failed updates are logged

## Security Implementation

### 1. Token Security

#### Storage Security
- **Current**: Tokens stored by oidc-client in browser storage
- **Recommendation**: Implement secure token storage with encryption
- **Consideration**: Token refresh and expiration handling

#### Token Validation
- **JWT Verification**: oidc-client validates JWT signatures
- **Expiration Check**: Automatic token refresh before expiration
- **Scope Validation**: Ensures requested permissions are granted

### 2. CSRF Protection

#### State Parameter
- **Purpose**: Prevents Cross-Site Request Forgery attacks
- **Implementation**: oidc-client generates and validates state
- **Validation**: State must match between request and callback

### 3. Redirect URI Validation

#### Security Measures
- **Exact Match**: Callback URL must exactly match Cognito configuration
- **HTTPS Requirement**: Production environments require HTTPS
- **Domain Validation**: Cognito validates redirect URI domain

## Error Handling and Logging

### 1. Authentication Errors

#### Common Error Scenarios
```typescript
// Network errors
catch (err) {
  console.error('Network error during authentication:', err);
}

// Configuration errors
catch (err) {
  console.error('OIDC configuration error:', err);
}

// User cancellation
catch (err) {
  console.error('User cancelled authentication:', err);
}
```

### 2. Custom Attribute Update Errors

#### Error Categories
- **Invalid Token**: Expired or malformed access token
- **Insufficient Permissions**: Token lacks required scope
- **Invalid Attributes**: Malformed attribute names or values
- **Network Issues**: AWS API communication failures

#### Error Handling Strategy
```typescript
try {
  const result = await this.cognitoClient.send(command);
  console.log('Custom attributes updated:', result);
} catch (err) {
  console.error('Error updating custom attributes:', err);
  
  // Specific error handling
  if (err.name === 'NotAuthorizedException') {
    console.error('Token expired or invalid');
  } else if (err.name === 'InvalidParameterException') {
    console.error('Invalid attribute format');
  }
}
```

## Performance Considerations

### 1. Token Management

#### Optimization Strategies
- **Lazy Loading**: Load user information only when needed
- **Token Caching**: Minimize token refresh operations
- **Background Refresh**: Refresh tokens before expiration

#### Memory Management
- **Token Cleanup**: Remove expired tokens from storage
- **User Object**: Clear user data on logout
- **Event Listeners**: Proper cleanup of OIDC event handlers

### 2. Network Optimization

#### API Calls
- **Batch Updates**: Group multiple attribute updates
- **Retry Logic**: Implement exponential backoff for failures
- **Connection Pooling**: Reuse AWS SDK connections

## Monitoring and Debugging

### 1. Debug Mode Configuration

#### OIDC Client Events
```typescript
// Enable detailed logging
this.userManager.events.addUserLoaded((user) => {
  console.log('User loaded:', user);
});

this.userManager.events.addUserUnloaded(() => {
  console.log('User unloaded');
});

this.userManager.events.addSilentRenewError((error) => {
  console.error('Silent renew error:', error);
});
```

#### AWS SDK Logging
```typescript
// Enable AWS SDK logging
const cognitoClient = new CognitoIdentityProviderClient({
  region: 'ap-south-1',
  logger: console
});
```

### 2. Performance Monitoring

#### Key Metrics
- Authentication success/failure rates
- Token refresh frequency
- Custom attribute update success rates
- API response times

#### Monitoring Tools
- Browser Developer Tools
- AWS CloudWatch
- Application Performance Monitoring (APM)
- Custom logging and metrics

## Deployment Considerations

### 1. Environment Configuration

#### Development vs Production
```typescript
// Development
const settings = {
  authority: 'https://cognito-idp.ap-south-1.amazonaws.com/ap-south-1_bs8cUkLpa',
  redirect_uri: 'http://localhost:4200/auth/callback'
};

// Production
const settings = {
  authority: 'https://cognito-idp.ap-south-1.amazonaws.com/PROD_USER_POOL_ID',
  redirect_uri: 'https://yourdomain.com/auth/callback'
};
```

### 2. Security Hardening

#### Production Requirements
- **HTTPS**: All communication must use HTTPS
- **CORS**: Proper Cross-Origin Resource Sharing configuration
- **Headers**: Security headers (HSTS, CSP, etc.)
- **Monitoring**: Comprehensive logging and alerting

#### Configuration Management
- **Environment Variables**: Sensitive data in environment variables
- **Secrets Management**: Use AWS Secrets Manager or similar
- **Configuration Validation**: Validate all configuration at startup

## Future Enhancements

### 1. Advanced Features

#### Multi-Factor Authentication
- TOTP integration
- SMS verification
- Hardware security keys

#### Role-Based Access Control
- Dynamic permission assignment
- Attribute-based access control
- Fine-grained permissions

### 2. Integration Capabilities

#### Additional Services
- AWS Lambda integration
- DynamoDB user data storage
- S3 file access control
- API Gateway integration

#### External Systems
- LDAP integration
- Active Directory synchronization
- Custom user stores
- Third-party identity providers

---

This technical architecture document provides a comprehensive understanding of the SSO implementation, enabling developers to maintain, extend, and troubleshoot the system effectively.
