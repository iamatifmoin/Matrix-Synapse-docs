# Matrix Synapse Integration - Technical Documentation

## Table of Contents

1. [Database Schema Changes](#database-schema-changes)
2. [Entity Modifications](#entity-modifications)
3. [Migration Details](#migration-details)
4. [Module Architecture](#module-architecture)
5. [Service Implementation](#service-implementation)
6. [Complete Flow: User Creation to Message Sending](#complete-flow-user-creation-to-message-sending)
7. [API Endpoints](#api-endpoints)
8. [Encryption Implementation](#encryption-implementation)
9. [Error Handling & Retry Logic](#error-handling--retry-logic)
10. [Integration Points](#integration-points)
11. [Matrix API Reference](#matrix-api-reference)
12. [Configuration Management](#configuration-management)
13. [Security Implementation](#security-implementation)
14. [Testing Considerations](#testing-considerations)

---

## Database Schema Changes

### Migration: `AddMatrixFields1767769359177`

**File**: `src/migrations/AddMatrixFields.ts`

This migration adds Matrix-related fields to three core tables in the database.

#### Users Table Changes

```sql
ALTER TABLE users 
ADD COLUMN matrix_user_id VARCHAR NULL,
ADD COLUMN matrix_access_token BYTEA NULL;
```

**Field Details**:

1. **`matrix_user_id`** (VARCHAR, nullable)
   - **Purpose**: Stores the Matrix user identifier
   - **Format**: `@sanitized_user_id:server_name`
   - **Example**: `@550e8400_e29b_41d4_a716_446655440000:localhost`
   - **Why VARCHAR**: Matrix user IDs are strings, not UUIDs
   - **Why Nullable**: User creation may fail, and we don't want to block user registration

2. **`matrix_access_token`** (BYTEA, nullable)
   - **Purpose**: Stores encrypted Matrix access token
   - **Format**: Binary data (encrypted buffer)
   - **Size**: Variable (typically ~200-500 bytes after encryption)
   - **Why BYTEA**: Stores binary encrypted data
   - **Why Nullable**: Token may not exist if Matrix user creation failed
   - **Why Encrypted**: Access tokens are sensitive credentials that must be protected

#### Jobs Table Changes

```sql
ALTER TABLE jobs 
ADD COLUMN matrix_room_id VARCHAR NULL,
ADD COLUMN matrix_room_alias VARCHAR NULL;
```

**Field Details**:

1. **`matrix_room_id`** (VARCHAR, nullable)
   - **Purpose**: Stores the Matrix room identifier
   - **Format**: `!room_id:server_name`
   - **Example**: `!abc123def456:localhost`
   - **Why VARCHAR**: Matrix room IDs are strings
   - **Why Nullable**: Room creation may fail, and draft jobs don't have rooms
   - **Usage**: Used to reference the room in Matrix API calls

2. **`matrix_room_alias`** (VARCHAR, nullable)
   - **Purpose**: Stores the human-readable room alias
   - **Format**: `#alias_name:server_name`
   - **Example**: `#job-550e8400-e29b-41d4-a716-446655440000:localhost`
   - **Why VARCHAR**: Room aliases are strings
   - **Why Nullable**: May not exist if room creation failed
   - **Usage**: Used for room discovery and user-friendly references

#### Organisations Table Changes

```sql
ALTER TABLE organisations 
ADD COLUMN matrix_room_id VARCHAR NULL;
```

**Field Details**:

1. **`matrix_room_id`** (VARCHAR, nullable)
   - **Purpose**: Stores the Matrix room identifier for organization
   - **Format**: `!room_id:server_name`
   - **Example**: `!xyz789ghi012:localhost`
   - **Why VARCHAR**: Matrix room IDs are strings
   - **Why Nullable**: Room creation may fail
   - **Note**: Organizations don't store room alias (not needed for org rooms)

### Migration Rollback

The migration includes a `down()` method for rollback:

```typescript
public async down(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.dropColumn('organisations', 'matrix_room_id');
  await queryRunner.dropColumn('jobs', 'matrix_room_alias');
  await queryRunner.dropColumn('jobs', 'matrix_room_id');
  await queryRunner.dropColumn('users', 'matrix_access_token');
  await queryRunner.dropColumn('users', 'matrix_user_id');
}
```

**Order Matters**: Columns are dropped in reverse order to avoid foreign key issues.

---

## Entity Modifications

### User Entity (`src/user/entity/user.entity.ts`)

**Added Fields**:

```typescript
@Column({ type: 'varchar', nullable: true })
matrix_user_id: string | null;

@Column({ type: 'bytea', nullable: true })
matrix_access_token: Buffer | null;
```

**TypeORM Decorators**:
- `@Column`: Marks as database column
- `type: 'varchar'`: String type for user ID
- `type: 'bytea'`: Binary type for encrypted token
- `nullable: true`: Allows NULL values

**TypeScript Types**:
- `string | null`: User ID can be null if Matrix account creation failed
- `Buffer | null`: Token stored as Buffer (binary data)

**Why These Types**:
- `string` for user ID: Matrix IDs are strings, not UUIDs
- `Buffer` for token: Encrypted data is binary, not text
- `null` allowed: Graceful degradation if Matrix is unavailable

### Job Entity (`src/jobs/entities/jobs.entity.ts`)

**Added Fields**:

```typescript
@Column({ type: 'varchar', nullable: true })
matrix_room_id: string | null;

@Column({ type: 'varchar', nullable: true })
matrix_room_alias: string | null;
```

**Usage in Code**:
- `matrix_room_id`: Used in API calls to reference the room
- `matrix_room_alias`: Used for room discovery and user-facing references

**Why Both Fields**:
- Room ID: Required for all Matrix API operations
- Room Alias: Provides human-readable reference and room discovery

### Organisation Entity (`src/organisations/entities/organisations.entity.ts`)

**Added Fields**:

```typescript
@Column({ type: 'varchar', nullable: true })
matrix_room_id: string | null;
```

**Note**: Organizations only store room ID, not alias (simpler use case).

---

## Migration Details

### Migration Class Structure

```typescript
export class AddMatrixFields1767769359177 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // Add columns
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Remove columns
  }
}
```

### Migration Execution

**When**: Runs automatically when database migrations are executed

**Command**: `npm run migration:run` or TypeORM migration command

**Idempotency**: Migration checks if columns exist before adding (TypeORM handles this)

### Migration Safety

- **Non-destructive**: Only adds columns, doesn't modify existing data
- **Nullable**: All new columns are nullable, so existing rows are unaffected
- **Rollback**: `down()` method allows safe rollback if needed

---

## Module Architecture

### MatrixModule Structure

**File**: `src/matrix/matrix.module.ts`

```typescript
@Module({
  imports: [
    HttpModule,                           // For HTTP requests to Matrix server
    TypeOrmModule.forFeature([User]),    // For accessing user repository
    forwardRef(() => AuthModule),         // For JWT authentication
  ],
  controllers: [MatrixController],
  providers: [MatrixService],
  exports: [MatrixService],              // Exported for use in other modules
})
export class MatrixModule {}
```

### Module Dependencies

**HttpModule**:
- Provides `HttpService` for making HTTP requests
- Used to communicate with Matrix Synapse server
- Configured via NestJS `@nestjs/axios`

**TypeOrmModule**:
- Provides `Repository<User>` for database access
- Used in `MatrixController` to fetch user data
- Only imports `User` entity (not Job or Organisation)

**AuthModule**:
- Provides JWT authentication guards
- Used in `MatrixController` to protect endpoints
- `forwardRef()` prevents circular dependency issues

### Module Exports

**MatrixService**:
- Exported so other modules can inject it
- Used by: `UserModule`, `JobsModule`, `OrganisationsModule`, `ApplicationsModule`

### Module Imports in Other Modules

**UserModule**:
```typescript
imports: [
  // ... other imports
  forwardRef(() => MatrixModule),
]
```

**JobsModule**:
```typescript
imports: [
  // ... other imports
  forwardRef(() => MatrixModule),
]
```

**OrganisationsModule**:
```typescript
imports: [
  // ... other imports
  forwardRef(() => MatrixModule),
]
```

**ApplicationsModule**:
```typescript
imports: [
  // ... other imports
  forwardRef(() => MatrixModule),
]
```

**Why `forwardRef()`**: Prevents circular dependency issues between modules.

---

## Service Implementation

### MatrixService Overview

**File**: `src/matrix/matrix.service.ts`

**Class Structure**:
```typescript
@Injectable()
export class MatrixService {
  private readonly logger: Logger;
  private readonly serverUrl: string;
  private readonly adminToken: string;
  private readonly serverName: string;

  constructor(private readonly httpService: HttpService) {
    // Initialize configuration
  }
}
```

### Configuration Initialization

```typescript
constructor(private readonly httpService: HttpService) {
  this.serverUrl = configService.getValue('MATRIX_SERVER_URL') || 'http://localhost:8008';
  this.adminToken = configService.getValue('MATRIX_ADMIN_TOKEN') || '';
  this.serverName = configService.getValue('MATRIX_SERVER_NAME') || 'localhost';
}
```

**Configuration Source**: `configService` (centralized configuration)

**Defaults**: Provides fallback values for development

### Core Methods

#### 1. User ID Sanitization

```typescript
private sanitizeUserId(userId: string): string {
  return userId.toLowerCase().replace(/[^a-z0-9_]/g, '_');
}
```

**Purpose**: Converts internal user ID to Matrix-compatible format

**Process**:
1. Convert to lowercase
2. Replace non-alphanumeric characters (except underscore) with underscore
3. Result: Valid Matrix localpart

**Example**: `"550e8400-e29b-41d4-a716-446655440000"` → `"550e8400_e29b_41d4_a716_446655440000"`

**Why**: Matrix user IDs have strict format requirements

#### 2. Retry Logic with Exponential Backoff

```typescript
private async executeWithRetry<T>(
  requestFn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelayMs: number = 1000,
): Promise<T> {
  let lastError: any;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error: any) {
      lastError = error;
      
      // Only retry on 429 (Too Many Requests)
      if (error.response?.status === 429 && attempt < maxRetries) {
        const headers = error.response?.headers || {};
        const retryAfter = headers['retry-after'] || headers['Retry-After'];
        let delayMs: number;
        
        if (retryAfter) {
          // Use Retry-After header if present
          delayMs = parseInt(retryAfter, 10) * 1000;
          this.logger.warn(
            `Rate limited (429). Retrying after ${retryAfter}s (attempt ${attempt + 1}/${maxRetries + 1})`,
          );
        } else {
          // Exponential backoff: baseDelayMs * 2^attempt
          delayMs = baseDelayMs * Math.pow(2, attempt);
          this.logger.warn(
            `Rate limited (429). Retrying after ${delayMs}ms (attempt ${attempt + 1}/${maxRetries + 1})`,
          );
        }
        
        await this.sleep(delayMs);
        continue;
      }
      
      // For non-429 errors or if we've exhausted retries, throw immediately
      throw error;
    }
  }
  
  throw lastError;
}
```

**Features**:
- Retries only on HTTP 429 (rate limiting)
- Respects `Retry-After` header if present
- Falls back to exponential backoff (1s, 2s, 4s)
- Maximum 3 retries (4 total attempts)
- Logs retry attempts for monitoring

**Why**: Matrix servers may rate limit requests; retry logic prevents failures

#### 3. Sleep Helper

```typescript
private sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

**Purpose**: Creates delay for retry logic

**Usage**: Used in retry mechanism to wait before retrying

#### 4. Create User

```typescript
async createUser(
  userId: string,
  email: string,
  displayName?: string,
): Promise<{ matrixUserId: string; accessToken: string }> {
  try {
    // 1. Sanitize user ID
    const sanitizedUserId = this.sanitizeUserId(userId);
    const matrixUserId = `@${sanitizedUserId}:${this.serverName}`;
    
    // 2. Generate random password
    const password = crypto.randomBytes(32).toString('hex');

    // 3. Create user via Admin API (with retry)
    try {
      await this.executeWithRetry(async () => {
        return await firstValueFrom(
          this.httpService.put(
            `${this.serverUrl}/_synapse/admin/v2/users/${encodeURIComponent(matrixUserId)}`,
            {
              password,
              displayname: displayName || email,
              admin: false,
            },
            {
              headers: {
                Authorization: `Bearer ${this.adminToken}`,
                'Content-Type': 'application/json',
              },
            },
          ),
        );
      });
    } catch (createError: any) {
      // 4. Handle existing user (409 conflict)
      if (createError.response?.status === 409) {
        this.logger.warn(`Matrix user ${matrixUserId} already exists, resetting password`);
        await this.executeWithRetry(async () => {
          return await firstValueFrom(
            this.httpService.put(
              `${this.serverUrl}/_synapse/admin/v2/users/${encodeURIComponent(matrixUserId)}`,
              {
                password,
                displayname: displayName || email,
              },
              {
                headers: {
                  Authorization: `Bearer ${this.adminToken}`,
                  'Content-Type': 'application/json',
                },
              },
            ),
          );
        });
      } else {
        throw createError;
      }
    }

    // 5. Login to get access token (with retry)
    const loginResponse = await this.executeWithRetry(async () => {
      return await firstValueFrom(
        this.httpService.post(
          `${this.serverUrl}/_matrix/client/v3/login`,
          {
            type: 'm.login.password',
            identifier: {
              type: 'm.id.user',
              user: matrixUserId,
            },
            password,
          },
        ),
      );
    });

    // 6. Return credentials
    return {
      matrixUserId,
      accessToken: loginResponse.data.access_token,
    };
  } catch (error: any) {
    this.logger.error(`Failed to create Matrix user: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Flow**:
1. Sanitize user ID → Generate Matrix user ID
2. Generate random 32-byte hex password
3. Create user via Admin API (PUT request)
4. Handle 409 conflict (user exists) → Reset password
5. Login via Client API (POST request)
6. Return user ID and access token

**Error Handling**:
- 409 Conflict: User exists → Reset password and continue
- Other errors: Log and throw

**Why Two API Calls**:
- Admin API: Creates user account
- Client API: Gets access token for the user

#### 5. Create Job Room

```typescript
async createJobRoom(
  creatorAccessToken: string,
  jobId: string,
  jobTitle: string,
  companyName: string,
): Promise<{ roomId: string; roomAlias: string }> {
  try {
    const roomAlias = `#job-${jobId}:${this.serverName}`;
    const name = `Job: ${jobTitle} - ${companyName}`;

    const response = await firstValueFrom(
      this.httpService.post(
        `${this.serverUrl}/_matrix/client/v3/createRoom`,
        {
          room_alias_name: `job-${jobId}`,
          name,
          topic: `Discussion room for job: ${jobTitle}`,
          visibility: 'public',
          preset: 'public_chat',
          invite: [],
        },
        {
          headers: {
            Authorization: `Bearer ${creatorAccessToken}`,
            'Content-Type': 'application/json',
          },
        },
      ),
    );

    return {
      roomId: response.data.room_id,
      roomAlias,
    };
  } catch (error: any) {
    this.logger.error(`Failed to create job room: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Parameters**:
- `creatorAccessToken`: Token of user creating the room (job creator)
- `jobId`: Internal job ID (used in alias)
- `jobTitle`: Job title (used in room name)
- `companyName`: Company name (used in room name)

**Room Configuration**:
- `room_alias_name`: Local part of alias (`job-{jobId}`)
- `name`: Display name
- `topic`: Room description
- `visibility`: `'public'` (discoverable)
- `preset`: `'public_chat'` (anyone can join)
- `invite`: Empty array (no initial invites)

**Returns**: Room ID and full alias

#### 6. Create Organization Room

```typescript
async createOrgRoom(
  creatorAccessToken: string,
  orgId: string,
  orgName: string,
): Promise<{ roomId: string }> {
  try {
    const roomAlias = `#org-${orgId}:${this.serverName}`;
    const name = `Organization: ${orgName}`;

    const response = await firstValueFrom(
      this.httpService.post(
        `${this.serverUrl}/_matrix/client/v3/createRoom`,
        {
          room_alias_name: `org-${orgId}`,
          name,
          topic: `Discussion room for organization: ${orgName}`,
          visibility: 'public',
          preset: 'public_chat',
          invite: [],
        },
        {
          headers: {
            Authorization: `Bearer ${creatorAccessToken}`,
            'Content-Type': 'application/json',
          },
        },
      ),
    );

    return {
      roomId: response.data.room_id,
    };
  } catch (error: any) {
    this.logger.error(`Failed to create org room: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Similar to Job Room**: Same structure, different naming convention

**Returns**: Only room ID (alias not needed for orgs)

#### 7. Add User to Room

```typescript
async addUserToRoom(
  inviterAccessToken: string,
  roomId: string,
  matrixUserId: string,
): Promise<void> {
  try {
    await firstValueFrom(
      this.httpService.post(
        `${this.serverUrl}/_matrix/client/v3/rooms/${encodeURIComponent(roomId)}/invite`,
        {
          user_id: matrixUserId,
        },
        {
          headers: {
            Authorization: `Bearer ${inviterAccessToken}`,
            'Content-Type': 'application/json',
          },
        },
      ),
    );
  } catch (error: any) {
    this.logger.error(
      `Failed to add user to room: ${error.message}`,
      error.stack,
    );
    throw error;
  }
}
```

**Parameters**:
- `inviterAccessToken`: Token of user doing the invite (job creator)
- `roomId`: Matrix room ID
- `matrixUserId`: Matrix user ID to add

**API Endpoint**: `POST /_matrix/client/v3/rooms/{roomId}/invite`

**Why Inviter Token**: Only room members with invite permission can invite others

#### 8. Remove User from Room

```typescript
async removeUserFromRoom(
  adminAccessToken: string,
  roomId: string,
  matrixUserId: string,
): Promise<void> {
  try {
    await firstValueFrom(
      this.httpService.post(
        `${this.serverUrl}/_matrix/client/v3/rooms/${encodeURIComponent(roomId)}/kick`,
        {
          user_id: matrixUserId,
          reason: 'Removed from job application',
        },
        {
          headers: {
            Authorization: `Bearer ${adminAccessToken}`,
            'Content-Type': 'application/json',
          },
        },
      ),
    );
  } catch (error: any) {
    this.logger.error(
      `Failed to remove user from room: ${error.message}`,
      error.stack,
    );
    throw error;
  }
}
```

**Parameters**:
- `adminAccessToken`: Admin token (not user token)
- `roomId`: Matrix room ID
- `matrixUserId`: Matrix user ID to remove

**API Endpoint**: `POST /_matrix/client/v3/rooms/{roomId}/kick`

**Why Admin Token**: Ensures removal succeeds even if inviter is no longer in room

**Reason**: Included in kick request for audit trail

#### 9. Get User Rooms

```typescript
async getUserRooms(accessToken: string): Promise<any[]> {
  try {
    const response = await firstValueFrom(
      this.httpService.get(
        `${this.serverUrl}/_matrix/client/v3/joined_rooms`,
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
          },
        },
      ),
    );

    return response.data.joined_rooms || [];
  } catch (error: any) {
    this.logger.error(`Failed to get user rooms: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Returns**: Array of room IDs the user has joined

**API Endpoint**: `GET /_matrix/client/v3/joined_rooms`

#### 10. Get Messages

```typescript
async getMessages(
  accessToken: string,
  roomId: string,
  limit: number = 50,
): Promise<any> {
  try {
    const response = await firstValueFrom(
      this.httpService.get(
        `${this.serverUrl}/_matrix/client/v3/rooms/${encodeURIComponent(roomId)}/messages`,
        {
          params: {
            dir: 'b',        // Backwards (most recent first)
            limit,           // Number of messages
          },
          headers: {
            Authorization: `Bearer ${accessToken}`,
          },
        },
      ),
    );

    return response.data;
  } catch (error: any) {
    this.logger.error(`Failed to get messages: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Parameters**:
- `accessToken`: User's access token
- `roomId`: Matrix room ID
- `limit`: Number of messages (default: 50)

**Query Parameters**:
- `dir: 'b'`: Backwards direction (newest first)
- `limit`: Number of messages to retrieve

**Returns**: Matrix API response with messages and pagination tokens

#### 11. Send Message

```typescript
async sendMessage(
  accessToken: string,
  roomId: string,
  message: string,
): Promise<string> {
  try {
    const txnId = `m${Date.now()}`;
    const response = await firstValueFrom(
      this.httpService.put(
        `${this.serverUrl}/_matrix/client/v3/rooms/${encodeURIComponent(roomId)}/send/m.room.message/${txnId}`,
        {
          msgtype: 'm.text',
          body: message,
        },
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
            'Content-Type': 'application/json',
          },
        },
      ),
    );

    return response.data.event_id;
  } catch (error: any) {
    this.logger.error(`Failed to send message: ${error.message}`, error.stack);
    throw error;
  }
}
```

**Parameters**:
- `accessToken`: User's access token
- `roomId`: Matrix room ID
- `message`: Message text content

**Transaction ID**: Generated as `m{timestamp}` for idempotency

**Message Format**:
- `msgtype: 'm.text'`: Plain text message
- `body`: Message content

**Returns**: Event ID of the sent message

**API Endpoint**: `PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}`

---

## Complete Flow: User Creation to Message Sending

### Flow 1: User Registration and Account Creation

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: User Registration Request                              │
│ POST /auth/register                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: UserService.create()                                    │
│ - Create user entity                                             │
│ - Hash email and phone                                           │
│ - Encrypt email and phone                                        │
│ - Hash password                                                  │
│ - Save user to database                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Matrix User Creation (if MatrixService available)      │
│                                                                   │
│ MatrixService.createUser():                                      │
│   3.1. Sanitize user ID                                          │
│        userId: "550e8400-e29b-41d4-a716-446655440000"           │
│        → "550e8400_e29b_41d4_a716_446655440000"                 │
│                                                                   │
│   3.2. Generate Matrix user ID                                  │
│        → "@550e8400_e29b_41d4_a716_446655440000:localhost"      │
│                                                                   │
│   3.3. Generate random password (32 bytes hex)                   │
│        → "a1b2c3d4e5f6..."                                      │
│                                                                   │
│   3.4. Create user via Admin API                                 │
│        PUT /_synapse/admin/v2/users/{matrixUserId}               │
│        Headers: Authorization: Bearer {adminToken}              │
│        Body: { password, displayname, admin: false }           │
│                                                                   │
│   3.5. Handle 409 Conflict (if user exists)                     │
│        → Reset password and update displayname                   │
│                                                                   │
│   3.6. Login to get access token                                 │
│        POST /_matrix/client/v3/login                             │
│        Body: { type: "m.login.password", identifier, password }  │
│        Response: { access_token: "syt_abc123..." }               │
│                                                                   │
│   3.7. Return credentials                                        │
│        { matrixUserId, accessToken }                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Encrypt and Store Access Token                          │
│                                                                   │
│ EncryptionUtil.encrypt(accessToken):                             │
│   4.1. Generate random IV (16 bytes)                             │
│   4.2. Create cipher (AES-256-GCM)                               │
│   4.3. Encrypt token                                             │
│   4.4. Get authentication tag                                    │
│   4.5. Concatenate: [IV][TAG][CIPHERTEXT]                        │
│                                                                   │
│ Update user entity:                                              │
│   - user.matrix_user_id = matrixUserId                           │
│   - user.matrix_access_token = encryptedBuffer                   │
│   - Save to database                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Return User Object                                       │
│ { id, email, matrix_user_id, ... }                               │
└─────────────────────────────────────────────────────────────────┘
```

**Error Handling**:
- If Matrix user creation fails → Log error, continue without Matrix account
- User is still created successfully
- Matrix fields remain NULL

### Flow 2: Job Creation and Room Creation

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Create Job Request                                      │
│ POST /jobs                                                      │
│ Body: { title, description, status: "open", ... }              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: JobsService.create()                                     │
│ - Create job entity                                              │
│ - Set relationships (company, category, skills)                  │
│ - Save job to database                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Check if Room Should Be Created                          │
│ if (job.status === 'open' &&                                     │
│     matrixService &&                                             │
│     !job.matrix_room_id)                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │ (true)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Get Job Creator                                          │
│ userRepository.findOne({ where: { id: createdBy } })           │
│ Check: creator.matrix_access_token exists                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Decrypt Access Token                                     │
│                                                                   │
│ EncryptionUtil.decrypt(creator.matrix_access_token):             │
│   5.1. Extract IV (first 16 bytes)                               │
│   5.2. Extract TAG (next 16 bytes)                               │
│   5.3. Extract CIPHERTEXT (remaining bytes)                       │
│   5.4. Create decipher                                           │
│   5.5. Set authentication tag                                    │
│   5.6. Decrypt and return string                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: Create Job Room                                          │
│                                                                   │
│ MatrixService.createJobRoom():                                   │
│   6.1. Generate room alias                                       │
│        → "#job-{jobId}:localhost"                                │
│                                                                   │
│   6.2. Generate room name                                        │
│        → "Job: {jobTitle} - {companyName}"                       │
│                                                                   │
│   6.3. Create room via Client API                                 │
│        POST /_matrix/client/v3/createRoom                       │
│        Headers: Authorization: Bearer {creatorAccessToken}       │
│        Body: {                                                   │
│          room_alias_name: "job-{jobId}",                         │
│          name: "Job: ...",                                       │
│          topic: "Discussion room for job: ...",                 │
│          visibility: "public",                                   │
│          preset: "public_chat",                                  │
│          invite: []                                              │
│        }                                                          │
│                                                                   │
│   6.4. Extract room ID from response                              │
│        → "!abc123def456:localhost"                                │
│                                                                   │
│   6.5. Return { roomId, roomAlias }                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 7: Store Room Information                                  │
│ job.matrix_room_id = room.roomId                                 │
│ job.matrix_room_alias = room.roomAlias                           │
│ jobsRepository.save(job)                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 8: Return Job Object                                        │
│ { id, title, matrix_room_id, matrix_room_alias, ... }            │
└─────────────────────────────────────────────────────────────────┘
```

**Error Handling**:
- If room creation fails → Log error, continue without room
- Job is still created successfully
- `matrix_room_id` remains NULL

### Flow 3: Application Status Change and Room Membership

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Update Application Request                               │
│ PATCH /applications/{id}                                         │
│ Body: { status: "accepted" }                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: ApplicationsService.update()                             │
│ - Load application with relations (job, appliedByUser)          │
│ - Store old status                                               │
│ - Update application status                                      │
│ - Save to database                                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Check if Room Membership Should Change                   │
│ if (matrixService &&                                             │
│     updated.job?.matrix_room_id &&                              │
│     user.matrix_user_id)                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │ (true)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Check Status Change Direction                           │
│                                                                   │
│ Case A: Status changed to "accepted"                             │
│   if (updateDto.status === 'accepted' &&                         │
│       oldStatus !== 'accepted')                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5A: Add User to Room                                        │
│                                                                   │
│   5A.1. Get job creator                                          │
│         userRepository.findOne({ where: { id: job.created_by } }) │
│                                                                   │
│   5A.2. Decrypt creator's access token                           │
│         EncryptionUtil.decrypt(creator.matrix_access_token)      │
│                                                                   │
│   5A.3. Add user to room                                         │
│         MatrixService.addUserToRoom():                           │
│         POST /_matrix/client/v3/rooms/{roomId}/invite            │
│         Headers: Authorization: Bearer {inviterAccessToken}      │
│         Body: { user_id: user.matrix_user_id }                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Case B: Status changed from "accepted" to "rejected"/"applied"   │
│   if ((updateDto.status === 'rejected' ||                        │
│        updateDto.status === 'applied') &&                        │
│       oldStatus === 'accepted')                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5B: Remove User from Room                                   │
│                                                                   │
│   5B.1. Get admin token                                          │
│         configService.getValue('MATRIX_ADMIN_TOKEN')             │
│                                                                   │
│   5B.2. Remove user from room                                    │
│         MatrixService.removeUserFromRoom():                       │
│         POST /_matrix/client/v3/rooms/{roomId}/kick               │
│         Headers: Authorization: Bearer {adminToken}              │
│         Body: {                                                  │
│           user_id: user.matrix_user_id,                          │
│           reason: "Removed from job application"                  │
│         }                                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: Return Updated Application                               │
│ { id, status: "accepted", job, appliedByUser, ... }               │
└─────────────────────────────────────────────────────────────────┘
```

**Error Handling**:
- If room membership update fails → Log error, continue
- Application update still succeeds
- Room membership may be out of sync (can be manually corrected)

### Flow 4: Sending a Message

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Send Message Request                                     │
│ POST /matrix/rooms/{roomId}/messages                            │
│ Headers: Authorization: Bearer {jwtToken}                       │
│ Body: { message: "Hello, this is a test message" }              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: MatrixController.sendMessage()                           │
│ - Extract user ID from JWT token                                 │
│ - Load user from database                                        │
│ - Check: user.matrix_access_token exists                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Decrypt Access Token                                     │
│                                                                   │
│ Handle buffer format:                                            │
│   if (Buffer.isBuffer(user.matrix_access_token))                 │
│     encryptedBuffer = user.matrix_access_token                   │
│   else                                                           │
│     encryptedBuffer = Buffer.from(                               │
│       user.matrix_access_token, 'base64'                          │
│     )                                                            │
│                                                                   │
│ EncryptionUtil.decrypt(encryptedBuffer):                         │
│   → "syt_abc123def456..."                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Send Message via Matrix API                              │
│                                                                   │
│ MatrixService.sendMessage():                                     │
│   4.1. Generate transaction ID                                   │
│        txnId = `m${Date.now()}`                                  │
│        → "m1704067200000"                                        │
│                                                                   │
│   4.2. Send message                                              │
│        PUT /_matrix/client/v3/rooms/{roomId}/                     │
│              send/m.room.message/{txnId}                         │
│        Headers: Authorization: Bearer {accessToken}              │
│        Body: {                                                   │
│          msgtype: "m.text",                                      │
│          body: "Hello, this is a test message"                    │
│        }                                                          │
│                                                                   │
│   4.3. Extract event ID from response                             │
│        → "$abc123def456:localhost"                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Return Event ID                                          │
│ { eventId: "$abc123def456:localhost" }                           │
└─────────────────────────────────────────────────────────────────┘
```

**Error Handling**:
- If access token missing → Throw error (user doesn't have Matrix account)
- If Matrix API fails → Throw error (message not sent)
- Frontend should handle errors gracefully

---

## API Endpoints

### MatrixController

**File**: `src/matrix/matrix.controller.ts`

**Base Path**: `/matrix`

**Authentication**: All endpoints require JWT authentication via `JwtAuthGuard`

#### 1. GET /matrix/token

**Purpose**: Retrieve current user's Matrix access token

**Authentication**: Required (JWT)

**Request**:
```
GET /matrix/token
Headers:
  Authorization: Bearer {jwtToken}
```

**Response**:
```json
{
  "accessToken": "syt_abc123def456..." // or null
}
```

**Implementation**:
```typescript
@UseGuards(JwtAuthGuard)
@Get('token')
async getToken(@Req() req: any) {
  const userId = req.user?.id || req.user?.sub;
  const user = await this.userRepo.findOne({ where: { id: userId } });

  if (!user || !user.matrix_access_token) {
    return { accessToken: null };
  }

  const encryptedBuffer = Buffer.isBuffer(user.matrix_access_token)
    ? user.matrix_access_token
    : Buffer.from(user.matrix_access_token, 'base64');

  const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

  return { accessToken };
}
```

**Error Cases**:
- User not found → Returns `{ accessToken: null }`
- No Matrix token → Returns `{ accessToken: null }`

#### 2. GET /matrix/rooms

**Purpose**: Get all rooms the user has joined

**Authentication**: Required (JWT)

**Request**:
```
GET /matrix/rooms
Headers:
  Authorization: Bearer {jwtToken}
```

**Response**:
```json
{
  "rooms": [
    "!abc123def456:localhost",
    "!xyz789ghi012:localhost"
  ]
}
```

**Implementation**:
```typescript
@UseGuards(JwtAuthGuard)
@Get('rooms')
async getRooms(@Req() req: any) {
  const userId = req.user?.id || req.user?.sub;
  const user = await this.userRepo.findOne({ where: { id: userId } });

  if (!user || !user.matrix_access_token) {
    return { rooms: [] };
  }

  const encryptedBuffer = Buffer.isBuffer(user.matrix_access_token)
    ? user.matrix_access_token
    : Buffer.from(user.matrix_access_token, 'base64');

  const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

  const rooms = await this.matrixService.getUserRooms(accessToken);
  return { rooms };
}
```

**Error Cases**:
- User not found → Returns `{ rooms: [] }`
- No Matrix token → Returns `{ rooms: [] }`
- Matrix API error → Throws error

#### 3. GET /matrix/rooms/:roomId/messages

**Purpose**: Retrieve messages from a room

**Authentication**: Required (JWT)

**Request**:
```
GET /matrix/rooms/{roomId}/messages?limit=50
Headers:
  Authorization: Bearer {jwtToken}
Query Parameters:
  limit (optional, default: 50): Number of messages to retrieve
```

**Response**: Matrix API response format:
```json
{
  "chunk": [
    {
      "type": "m.room.message",
      "content": {
        "msgtype": "m.text",
        "body": "Hello, world!"
      },
      "event_id": "$abc123:localhost",
      "sender": "@user123:localhost",
      "origin_server_ts": 1704067200000
    }
  ],
  "start": "t123-456",
  "end": "t789-012"
}
```

**Implementation**:
```typescript
@UseGuards(JwtAuthGuard)
@Get('rooms/:roomId/messages')
async getMessages(
  @Param('roomId') roomId: string,
  @Query('limit') limit: string,
  @Req() req: any,
) {
  const userId = req.user?.id || req.user?.sub;
  const user = await this.userRepo.findOne({ where: { id: userId } });

  if (!user || !user.matrix_access_token) {
    return { messages: [] };
  }

  const encryptedBuffer = Buffer.isBuffer(user.matrix_access_token)
    ? user.matrix_access_token
    : Buffer.from(user.matrix_access_token, 'base64');

  const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

  const messages = await this.matrixService.getMessages(
    accessToken,
    roomId,
    parseInt(limit, 10) || 50,
  );

  return { messages };
}
```

**Error Cases**:
- User not found → Returns `{ messages: [] }`
- No Matrix token → Returns `{ messages: [] }`
- Invalid room ID → Matrix API returns error
- User not in room → Matrix API returns 403 Forbidden

#### 4. POST /matrix/rooms/:roomId/messages

**Purpose**: Send a message to a room

**Authentication**: Required (JWT)

**Request**:
```
POST /matrix/rooms/{roomId}/messages
Headers:
  Authorization: Bearer {jwtToken}
  Content-Type: application/json
Body:
  {
    "message": "Hello, this is a test message"
  }
```

**Response**:
```json
{
  "eventId": "$abc123def456:localhost"
}
```

**Implementation**:
```typescript
@UseGuards(JwtAuthGuard)
@Post('rooms/:roomId/messages')
async sendMessage(
  @Param('roomId') roomId: string,
  @Body() body: { message: string },
  @Req() req: any,
) {
  const userId = req.user?.id || req.user?.sub;
  const user = await this.userRepo.findOne({ where: { id: userId } });

  if (!user || !user.matrix_access_token) {
    throw new Error('Matrix access token not found');
  }

  const encryptedBuffer = Buffer.isBuffer(user.matrix_access_token)
    ? user.matrix_access_token
    : Buffer.from(user.matrix_access_token, 'base64');

  const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

  const eventId = await this.matrixService.sendMessage(
    accessToken,
    roomId,
    body.message,
  );

  return { eventId };
}
```

**Error Cases**:
- User not found → Throws error
- No Matrix token → Throws error
- Invalid room ID → Matrix API returns error
- User not in room → Matrix API returns 403 Forbidden
- Message too long → Matrix API returns error

---

## Encryption Implementation

### EncryptionUtil Class

**File**: `src/utils/crypto/encryption.util.ts`

**Algorithm**: AES-256-GCM (Advanced Encryption Standard with Galois/Counter Mode)

**Key Requirements**:
- 32 bytes (256 bits)
- Stored as hex string in environment variable
- Must be kept secret

### Encryption Process

```typescript
static encrypt(value: string): Buffer {
  // 1. Generate random initialization vector (IV)
  const iv = crypto.randomBytes(16);  // 16 bytes = 128 bits

  // 2. Create cipher with algorithm, key, and IV
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);

  // 3. Encrypt the value
  const encrypted = Buffer.concat([
    cipher.update(value, 'utf8'),  // Convert string to bytes and encrypt
    cipher.final(),                 // Finalize encryption
  ]);

  // 4. Get authentication tag (prevents tampering)
  const tag = cipher.getAuthTag();  // 16 bytes

  // 5. Concatenate: [IV (16 bytes)][TAG (16 bytes)][CIPHERTEXT (variable)]
  return Buffer.concat([iv, tag, encrypted]);
}
```

**Output Format**:
```
[IV: 16 bytes][TAG: 16 bytes][CIPHERTEXT: variable length]
```

**Why GCM Mode**:
- Provides authenticated encryption (detects tampering)
- Includes authentication tag
- Industry standard for secure encryption

### Decryption Process

```typescript
static decrypt(encrypted: Buffer): string {
  // 1. Extract components
  const iv = encrypted.slice(0, 16);        // First 16 bytes
  const tag = encrypted.slice(16, 32);       // Next 16 bytes
  const content = encrypted.slice(32);       // Remaining bytes

  // 2. Create decipher
  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv);

  // 3. Set authentication tag (verifies integrity)
  decipher.setAuthTag(tag);

  // 4. Decrypt
  const decrypted = Buffer.concat([
    decipher.update(content),
    decipher.final(),
  ]);

  // 5. Convert to string
  return decrypted.toString('utf8');
}
```

**Error Handling**:
- If tag doesn't match → Throws error (tampering detected)
- If IV is wrong → Decryption fails
- If key is wrong → Decryption fails

### Buffer Handling in Code

**Database Storage**:
- Stored as `BYTEA` in PostgreSQL
- TypeORM maps to `Buffer` type in TypeScript

**Retrieval from Database**:
```typescript
// TypeORM may return Buffer or base64 string
const encryptedBuffer = Buffer.isBuffer(user.matrix_access_token)
  ? user.matrix_access_token
  : Buffer.from(user.matrix_access_token, 'base64');
```

**Why This Handling**:
- TypeORM may serialize Buffer to base64 in some contexts
- Need to handle both cases for compatibility

---

## Error Handling & Retry Logic

### Retry Strategy

**When Retries Happen**:
- Only on HTTP 429 (Too Many Requests)
- Maximum 3 retries (4 total attempts)
- Uses exponential backoff or `Retry-After` header

**Retry Logic Flow**:
```
Attempt 1 → 429 Error → Wait → Retry
Attempt 2 → 429 Error → Wait → Retry
Attempt 3 → 429 Error → Wait → Retry
Attempt 4 → Success or Throw Error
```

**Delay Calculation**:
1. Check for `Retry-After` header (preferred)
2. If present: Use header value (in seconds, convert to milliseconds)
3. If absent: Use exponential backoff
   - Attempt 1: 1000ms (1 second)
   - Attempt 2: 2000ms (2 seconds)
   - Attempt 3: 4000ms (4 seconds)

### Graceful Degradation

**Principle**: Matrix failures should never break core functionality

**Implementation Pattern**:
```typescript
try {
  // Matrix operation
  await this.matrixService.createUser(...);
} catch (error) {
  // Log error but don't throw
  this.logger.error(`Failed to create Matrix user: ${error.message}`);
  // Continue without Matrix
}
```

**Applied To**:
- User creation → User still created
- Room creation → Job/org still created
- Room membership → Application still updated

**Benefits**:
- Platform remains functional if Matrix is down
- Users can still use core features
- Matrix issues don't cascade to other features

### Error Logging

**Logging Strategy**:
- All Matrix errors are logged with context
- Includes: operation type, entity ID, error message, stack trace
- Uses NestJS Logger for consistent formatting

**Example Log Messages**:
```
[MatrixService] Failed to create Matrix user for abc123: Connection timeout
[MatrixService] Failed to create Matrix room for job xyz789: Rate limit exceeded
[ApplicationsService] Failed to update Matrix room membership: User not found
```

---

## Integration Points

### UserService Integration

**File**: `src/user/user.service.ts`

**Injection**:
```typescript
constructor(
  // ... other dependencies
  private readonly matrixService?: MatrixService,
) {}
```

**Integration Point**: `create()` method

**Code**:
```typescript
const savedUser = await repo.save(user);

// Create Matrix user (don't fail if this fails)
if (this.matrixService) {
  try {
    const displayName = savedUser.display_name || 
      `${savedUser.first_name} ${savedUser.last_name}`.trim();
    const matrixUser = await this.matrixService.createUser(
      savedUser.id,
      dto.email,
      displayName,
    );

    savedUser.matrix_user_id = matrixUser.matrixUserId;
    savedUser.matrix_access_token = EncryptionUtil.encrypt(matrixUser.accessToken);
    await repo.save(savedUser);
  } catch (error) {
    this.logger.error(`Failed to create Matrix user: ${error.message}`, error.stack);
  }
}

return savedUser;
```

**Why Optional**: `matrixService?: MatrixService` allows system to work without Matrix

### JobsService Integration

**File**: `src/jobs/jobs.service.ts`

**Integration Points**: `create()` and `update()` methods

**Code** (in `create()`):
```typescript
// Create Matrix room if job is published (status is 'open')
if (job.status === 'open' && this.matrixService && !job.matrix_room_id) {
  try {
    const creator = await this.userRepository.findOne({
      where: { id: createdBy },
    });

    if (creator && creator.matrix_access_token) {
      const encryptedBuffer = Buffer.isBuffer(creator.matrix_access_token)
        ? creator.matrix_access_token
        : Buffer.from(creator.matrix_access_token, 'base64');
      const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

      const companyName = job.company?.display_name || 'Company';
      const room = await this.matrixService.createJobRoom(
        accessToken,
        job.id,
        job.title,
        companyName,
      );

      job.matrix_room_id = room.roomId;
      job.matrix_room_alias = room.roomAlias;
      await this.jobsRepository.save(job);
    }
  } catch (error) {
    this.logger.error(
      `Failed to create Matrix room for job ${job.id}: ${error.message}`,
      error.stack,
    );
  }
}
```

**Similar Code**: In `update()` method when status changes to 'open'

### OrganisationsService Integration

**File**: `src/organisations/organisations.service.ts`

**Integration Point**: `create()` method

**Code**:
```typescript
// Create Matrix room for organization
if (this.matrixService && !savedOrg.matrix_room_id) {
  try {
    const creator = await this.userRepo.findOne({
      where: { id: userId },
    });

    if (creator && creator.matrix_access_token) {
      const encryptedBuffer = Buffer.isBuffer(creator.matrix_access_token)
        ? creator.matrix_access_token
        : Buffer.from(creator.matrix_access_token, 'base64');
      const accessToken = EncryptionUtil.decrypt(encryptedBuffer);

      const orgName = savedOrg.display_name || savedOrg.legal_name || 'Organization';
      const room = await this.matrixService.createOrgRoom(
        accessToken,
        savedOrg.id,
        orgName,
      );

      savedOrg.matrix_room_id = room.roomId;
      await repo.save(savedOrg);
    }
  } catch (error) {
    this.logger.error(
      `Failed to create Matrix room for org ${savedOrg.id}: ${error.message}`,
      error.stack,
    );
  }
}
```

### ApplicationsService Integration

**File**: `src/applications/applications.service.ts`

**Integration Points**: `update()` and `remove()` methods

**Code** (in `update()`):
```typescript
// Handle Matrix room membership changes
if (this.matrixService && updated.job?.matrix_room_id) {
  try {
    const job = updated.job;
    const user = updated.appliedByUser;

    if (!user || !user.matrix_user_id) {
      return updated;
    }

    const adminToken = configService.getValue('MATRIX_ADMIN_TOKEN') || '';

    // If status changed to 'accepted', add user to room
    if (
      updateDto.status === 'accepted' &&
      oldStatus !== 'accepted' &&
      job.matrix_room_id
    ) {
      const creator = await this.userRepo.findOne({
        where: { id: job.created_by },
      });

      if (creator && creator.matrix_access_token) {
        const encryptedBuffer = Buffer.isBuffer(creator.matrix_access_token)
          ? creator.matrix_access_token
          : Buffer.from(creator.matrix_access_token, 'base64');
        const inviterAccessToken = EncryptionUtil.decrypt(encryptedBuffer);

        await this.matrixService.addUserToRoom(
          inviterAccessToken,
          job.matrix_room_id,
          user.matrix_user_id,
        );
      }
    }

    // If status changed to 'rejected' or 'applied', remove user from room
    if (
      (updateDto.status === 'rejected' || updateDto.status === 'applied') &&
      oldStatus === 'accepted' &&
      job.matrix_room_id
    ) {
      await this.matrixService.removeUserFromRoom(
        adminToken,
        job.matrix_room_id,
        user.matrix_user_id,
      );
    }
  } catch (error) {
    this.logger.error(
      `Failed to update Matrix room membership: ${error.message}`,
      error.stack,
    );
  }
}
```

**Code** (in `remove()`):
```typescript
if (application && this.matrixService && application.job?.matrix_room_id) {
  try {
    const user = application.appliedByUser;
    if (user && user.matrix_user_id && application.status === 'accepted') {
      const adminToken = configService.getValue('MATRIX_ADMIN_TOKEN') || '';
      await this.matrixService.removeUserFromRoom(
        adminToken,
        application.job.matrix_room_id,
        user.matrix_user_id,
      );
    }
  } catch (error) {
    this.logger.error(
      `Failed to remove user from Matrix room: ${error.message}`,
      error.stack,
    );
  }
}
```

---

## Matrix API Reference

### Admin API Endpoints

#### Create/Update User
```
PUT /_synapse/admin/v2/users/{userId}
Authorization: Bearer {adminToken}
Content-Type: application/json

Body:
{
  "password": "random_hex_string",
  "displayname": "User Display Name",
  "admin": false
}

Response: 200 OK (or 409 Conflict if user exists)
```

**Used For**: Creating Matrix user accounts

**Authentication**: Admin token (not user token)

#### Update Existing User
```
PUT /_synapse/admin/v2/users/{userId}
Authorization: Bearer {adminToken}
Content-Type: application/json

Body:
{
  "password": "new_random_hex_string",
  "displayname": "Updated Display Name"
}

Response: 200 OK
```

**Used For**: Resetting password when user already exists (409 conflict)

### Client API Endpoints

#### Login
```
POST /_matrix/client/v3/login
Content-Type: application/json

Body:
{
  "type": "m.login.password",
  "identifier": {
    "type": "m.id.user",
    "user": "@user_id:server_name"
  },
  "password": "user_password"
}

Response:
{
  "access_token": "syt_abc123...",
  "device_id": "device_id",
  "user_id": "@user_id:server_name"
}
```

**Used For**: Getting access token after user creation

**Authentication**: None (uses password)

#### Create Room
```
POST /_matrix/client/v3/createRoom
Authorization: Bearer {accessToken}
Content-Type: application/json

Body:
{
  "room_alias_name": "room_alias",
  "name": "Room Display Name",
  "topic": "Room Description",
  "visibility": "public",
  "preset": "public_chat",
  "invite": []
}

Response:
{
  "room_id": "!room_id:server_name",
  "room_alias": "#room_alias:server_name"
}
```

**Used For**: Creating job and organization rooms

**Authentication**: User access token

#### Invite User
```
POST /_matrix/client/v3/rooms/{roomId}/invite
Authorization: Bearer {accessToken}
Content-Type: application/json

Body:
{
  "user_id": "@user_id:server_name"
}

Response: 200 OK
```

**Used For**: Adding users to job rooms

**Authentication**: Inviter's access token (must be room member)

#### Kick User
```
POST /_matrix/client/v3/rooms/{roomId}/kick
Authorization: Bearer {adminToken}
Content-Type: application/json

Body:
{
  "user_id": "@user_id:server_name",
  "reason": "Removed from job application"
}

Response: 200 OK
```

**Used For**: Removing users from rooms

**Authentication**: Admin token (ensures success even if inviter left)

#### Get Joined Rooms
```
GET /_matrix/client/v3/joined_rooms
Authorization: Bearer {accessToken}

Response:
{
  "joined_rooms": [
    "!room1:server_name",
    "!room2:server_name"
  ]
}
```

**Used For**: Listing user's rooms

**Authentication**: User access token

#### Get Messages
```
GET /_matrix/client/v3/rooms/{roomId}/messages?dir=b&limit=50
Authorization: Bearer {accessToken}

Query Parameters:
- dir: "b" (backwards) or "f" (forwards)
- limit: Number of messages (default: 10)

Response:
{
  "chunk": [...messages...],
  "start": "pagination_token",
  "end": "pagination_token"
}
```

**Used For**: Retrieving room messages

**Authentication**: User access token

#### Send Message
```
PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}
Authorization: Bearer {accessToken}
Content-Type: application/json

Body:
{
  "msgtype": "m.text",
  "body": "Message content"
}

Response:
{
  "event_id": "$event_id:server_name"
}
```

**Used For**: Sending messages to rooms

**Authentication**: User access token

**Transaction ID**: Unique ID for idempotency (format: `m{timestamp}`)

---

## Configuration Management

### Environment Variables

**Required Variables**:

```bash
# Matrix Server Configuration
MATRIX_SERVER_URL=http://localhost:8008
MATRIX_ADMIN_TOKEN=your_admin_token_here
MATRIX_SERVER_NAME=localhost

# Encryption Key (32 bytes hex)
ENCRYPTION_KEY=your_32_byte_hex_key_here
```

### Configuration Loading

**File**: Uses `configService` (centralized configuration)

**Loading in MatrixService**:
```typescript
this.serverUrl = configService.getValue('MATRIX_SERVER_URL') || 'http://localhost:8008';
this.adminToken = configService.getValue('MATRIX_ADMIN_TOKEN') || '';
this.serverName = configService.getValue('MATRIX_SERVER_NAME') || 'localhost';
```

**Defaults**: Provides fallback values for development

### Configuration Validation

**Current**: No explicit validation (relies on runtime errors)

**Recommendation**: Add validation on startup to catch missing configuration early

---

## Security Implementation

### Access Token Security

**Encryption**:
- Algorithm: AES-256-GCM
- Key: 32-byte hex string from environment
- Storage: Encrypted in database (BYTEA column)

**Why Encrypt**:
- Access tokens are sensitive credentials
- If database is compromised, tokens are still protected
- GCM mode provides authentication (tamper detection)

### Token Handling

**Never Logged**: Access tokens are never logged in plain text

**Decryption**: Only decrypted when needed for API calls

**Lifetime**: Tokens don't expire (Matrix long-lived tokens)

**Rotation**: Not currently implemented (would require re-login)

### API Security

**JWT Authentication**: All Matrix endpoints require valid JWT

**User Isolation**: Users can only access their own tokens

**Room Access**: Matrix server enforces room membership

**Admin Operations**: Use admin token (not user token) for removals

### Input Validation

**User ID Sanitization**: Prevents injection attacks

**Room ID Encoding**: URL-encoded in API calls

**Message Content**: No validation (Matrix server handles)

---

## Testing Considerations

### Unit Testing

**MatrixService**:
- Mock `HttpService` for API calls
- Test retry logic with 429 errors
- Test user ID sanitization
- Test error handling

**EncryptionUtil**:
- Test encryption/decryption roundtrip
- Test with various input lengths
- Test tamper detection (modified tag)

**Controllers**:
- Mock `MatrixService`
- Test JWT authentication
- Test error responses

### Integration Testing

**User Creation Flow**:
- Test Matrix user creation on registration
- Test graceful degradation if Matrix fails
- Test existing user handling (409 conflict)

**Room Creation Flow**:
- Test job room creation
- Test organization room creation
- Test room creation failure handling

**Room Membership Flow**:
- Test adding user to room
- Test removing user from room
- Test status change scenarios

### E2E Testing

**Full Flow**:
1. Register user → Verify Matrix account created
2. Create job → Verify room created
3. Apply to job → Verify no room access
4. Accept application → Verify room access
5. Send message → Verify message sent
6. Reject application → Verify room access removed

### Mock Matrix Server

**Options**:
- Use actual Matrix test server
- Mock Matrix API responses
- Use Matrix SDK test utilities

**Recommendation**: Use actual test Matrix server for integration tests

---

## Summary

This technical documentation covers:

1. **Database Schema**: All new fields and their purposes
2. **Entity Modifications**: TypeORM entity changes
3. **Service Implementation**: Complete MatrixService with all methods
4. **Complete Flows**: Step-by-step flows from user creation to messaging
5. **API Endpoints**: All REST endpoints with request/response formats
6. **Encryption**: AES-256-GCM implementation details
7. **Error Handling**: Retry logic and graceful degradation
8. **Integration Points**: How Matrix integrates with other services
9. **Matrix API Reference**: All Matrix endpoints used
10. **Security**: Token encryption and access control
11. **Testing**: Considerations for unit, integration, and E2E tests

The integration is designed to be:
- **Resilient**: Failures don't break core functionality
- **Secure**: Tokens encrypted, access controlled
- **Maintainable**: Clear separation of concerns, well-documented
- **Scalable**: Retry logic handles rate limiting, graceful degradation handles outages
