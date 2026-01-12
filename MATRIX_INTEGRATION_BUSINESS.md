# Matrix Chat Integration - Business & Product Guide

## Executive Summary

The platform integrates Matrix chat functionality to enable real-time communication around job postings and organizations. Every user automatically gets a chat account, and dedicated chat rooms are created for each published job and organization. This allows recruiters, candidates, and hiring managers to communicate directly within the platform.

---

## Table of Contents

1. [Overview](#overview)
2. [User Accounts](#user-accounts)
3. [Chat Rooms](#chat-rooms)
4. [Room Access & Membership](#room-access--membership)
5. [Message Features](#message-features)
6. [Room Lifecycle](#room-lifecycle)
7. [Data Storage & Privacy](#data-storage--privacy)
8. [Configuration & Setup](#configuration--setup)
9. [User Experience Flow](#user-experience-flow)
10. [Business Rules & Policies](#business-rules--policies)
11. [Troubleshooting & Support](#troubleshooting--support)

---

## Overview

### What is Matrix Chat?

Matrix is an open-source, decentralized chat protocol that powers real-time messaging. In this platform, Matrix provides:

- **Job Discussion Rooms**: Chat rooms for each published job where recruiters and hiring managers can discuss candidates
- **Organization Rooms**: Chat rooms for each organization to facilitate internal communication
- **Automatic Account Creation**: Every user in the system automatically gets a chat account
- **Secure Messaging**: All messages are encrypted and stored securely

### Key Benefits

- **Seamless Integration**: No separate sign-up required - chat accounts are created automatically
- **Contextual Communication**: Chat rooms are tied directly to jobs and organizations
- **Access Control**: Only relevant users can access specific job rooms
- **Persistent History**: All messages are saved and accessible to authorized users

---

## User Accounts

### Automatic Account Creation

**When**: Every time a new user registers or is created in the system

**What Happens**:
1. A chat account is automatically created for the user
2. The user receives a unique chat ID (e.g., `@user_123:company.com`)
3. An access token is generated and securely stored
4. The user can immediately start using chat features

**User Experience**:
- Users don't need to do anything - it happens automatically
- No separate sign-up or login required for chat
- If account creation fails, the user can still use the platform (chat just won't be available)

### Account Recovery

If a user's chat account already exists (e.g., from a previous registration attempt), the system automatically resets their password and updates their display name. This ensures users can always access their chat account.

---

## Chat Rooms

### Job Rooms

#### When Are Job Rooms Created?

Job rooms are automatically created when:
1. **A job is published** - When a job's status changes from "draft" to "open"
2. **A new job is created as "open"** - If a job is created directly with "open" status

**Important**: Job rooms are **only** created for published jobs (status = "open"). Draft jobs do not have chat rooms.

#### Room Details

**Room Name Format**: `Job: [Job Title] - [Company Name]`

**Example**: 
- Job Title: "Senior Software Engineer"
- Company: "Tech Corp"
- Room Name: "Job: Senior Software Engineer - Tech Corp"

**Room Description**: "Discussion room for job: [Job Title]"

**Room Type**: Public chat room (anyone with the link can join, but access is controlled by the platform)

**Room Alias**: Each room has a unique alias like `#job-abc123:company.com` that can be used to reference the room

#### Who Can Access Job Rooms?

**Initial Access**:
- The job creator (hiring manager/recruiter who posted the job) automatically has access

**Additional Access**:
- Recruiters whose applications are **accepted** for that job are automatically added
- Once added, they have full access to:
  - View all message history (including messages before they joined)
  - Send messages
  - See all participants

**Access Removal**:
- If a recruiter's application status changes from "accepted" to "rejected" or "applied", they are automatically removed from the room
- If an application is deleted, the recruiter is removed from the room (if they were previously accepted)

### Organization Rooms

#### When Are Organization Rooms Created?

Organization rooms are automatically created when:
- A new organization is created in the system

**Note**: Unlike job rooms, organization rooms are created immediately upon organization creation, regardless of status.

#### Room Details

**Room Name Format**: `Organization: [Organization Name]`

**Example**:
- Organization: "Acme Recruiting"
- Room Name: "Organization: Acme Recruiting"

**Room Description**: "Discussion room for organization: [Organization Name]"

**Room Type**: Public chat room

**Room Alias**: Each room has a unique alias like `#org-xyz789:company.com`

#### Who Can Access Organization Rooms?

Currently, organization rooms are created as public rooms. Access control is managed at the platform level - users who are members of the organization would have access through the platform's organization membership system.

---

## Room Access & Membership

### How Users Are Added to Rooms

#### Job Rooms

**Method**: Users are **directly added** (not invited) to job rooms

**Process**:
1. When a recruiter's application status changes to "accepted"
2. The system automatically adds them to the job's chat room
3. No invitation is sent - they are immediately added
4. The user can immediately see the room and all message history

**Who Adds Users?**
- The job creator's account is used to add users
- This ensures proper permissions and room management

#### Organization Rooms

Organization rooms are public, meaning:
- Anyone with the room alias can join
- Access is typically controlled through the platform's organization membership features

### How Users Are Removed from Rooms

**Job Rooms**:
- Users are automatically removed when:
  - Their application status changes from "accepted" to "rejected" or "applied"
  - Their application is deleted
- Removal uses administrative permissions to ensure it always succeeds
- Users lose access immediately upon removal

**Organization Rooms**:
- Currently, removal is not automatically managed
- Manual removal would be required if needed

### Message History Access

**Key Point**: When a user is added to a room, they can see **all previous messages** in that room, including messages sent before they joined.

**Implications**:
- New members have full context of the conversation
- All discussion history is preserved
- No information is lost when members are added

---

## Message Features

### Sending Messages

**Who Can Send**:
- Any user who has access to a room can send messages
- Messages are sent in real-time
- Messages are plain text (no rich formatting currently)

**Message Format**:
- Simple text messages
- Each message gets a unique ID for tracking

### Viewing Messages

**Message Retrieval**:
- Users can retrieve messages from any room they have access to
- Default view shows the 50 most recent messages
- Messages are displayed in reverse chronological order (newest first)
- Users can request more messages if needed

**Message History**:
- All messages are permanently stored
- Users can access full conversation history
- History is available as long as the user has room access

### Message Storage

**Where Messages Are Stored**:
- Messages are stored on the Matrix server (not in the platform's database)
- The Matrix server maintains a complete message history
- Messages are encrypted in transit and at rest on the Matrix server

**Retention Policy**:
- Messages are stored indefinitely
- There is no automatic deletion of messages
- Room deletion would remove all messages (currently rooms are not automatically deleted)

---

## Room Lifecycle

### Room Creation

**Job Rooms**:
- Created when job status becomes "open"
- Created only once per job
- If creation fails, the job is still created (chat just won't be available)

**Organization Rooms**:
- Created when organization is created
- Created only once per organization
- If creation fails, the organization is still created

### Room Status

**Active Rooms**:
- Rooms remain active as long as the job/organization exists
- Rooms are not automatically closed or archived

**Room Closure**:
- Currently, rooms are **not automatically closed**
- Rooms remain accessible even if:
  - A job is closed or put on hold
  - An organization is deactivated
- Manual closure would require administrative action

### Room Deletion

**Current Behavior**:
- Rooms are **not automatically deleted**
- Even if a job or organization is deleted from the platform, the Matrix room may still exist
- This ensures message history is preserved

**Future Considerations**:
- May want to implement automatic room closure when jobs are closed
- May want to archive rooms instead of deleting them
- May want to implement retention policies

---

## Data Storage & Privacy

### Where Data Is Stored

**User Chat Accounts**:
- User chat IDs are stored in the platform database
- Access tokens are encrypted and stored in the platform database
- Chat account passwords are managed by Matrix (not stored in platform)

**Room Information**:
- Room IDs are stored in the platform database (linked to jobs/organizations)
- Room aliases are stored in the platform database
- Room membership is managed by Matrix server

**Messages**:
- Messages are stored on the Matrix server
- Messages are not stored in the platform database
- Message content is encrypted on the Matrix server

### Data Security

**Encryption**:
- Access tokens are encrypted using AES-256-GCM encryption
- Messages are encrypted in transit (HTTPS)
- Messages are encrypted at rest on Matrix server

**Access Control**:
- Only users with valid access tokens can access rooms
- Room membership is controlled by the platform
- Users can only see rooms they have been added to

**Privacy**:
- Users cannot see rooms they don't have access to
- Message history is only visible to room members
- No cross-room message visibility

### Data Retention

**User Accounts**:
- Chat accounts persist as long as the user account exists
- If a user is deleted, their chat account may still exist on Matrix (manual cleanup may be needed)

**Rooms**:
- Rooms persist indefinitely
- Message history is preserved
- No automatic cleanup

---

## Configuration & Setup

### Required Configuration

To enable Matrix chat functionality, the following must be configured:

1. **Matrix Server URL**
   - The address of the Matrix server (e.g., `https://matrix.company.com`)
   - Default: `http://localhost:8008` (for development)

2. **Admin Token**
   - A special token that allows the platform to create user accounts
   - This is a Matrix server administrative token
   - Required for automatic user account creation

3. **Server Name**
   - The domain name of the Matrix server (e.g., `company.com`)
   - Used in user IDs and room aliases
   - Default: `localhost` (for development)

4. **Encryption Key**
   - A 32-byte encryption key for securing access tokens
   - Must be kept secret and secure
   - Used to encrypt/decrypt user access tokens

### Configuration Impact

**If Matrix is Not Configured**:
- The platform continues to function normally
- Chat features are simply unavailable
- No errors are shown to users
- Users can still use all other platform features

**If Matrix Configuration is Incorrect**:
- User account creation may fail (but users are still created)
- Room creation may fail (but jobs/orgs are still created)
- Chat features won't work, but platform remains functional

---

## User Experience Flow

### For New Users

1. **User Registers** → Chat account is automatically created
2. **User Logs In** → Can access chat features immediately
3. **User Applies to Job** → Application is created
4. **Application Accepted** → User is automatically added to job chat room
5. **User Opens Chat** → Can see job room and start messaging

### For Job Creators

1. **Create Job** → Job is created as draft
2. **Publish Job** → Chat room is automatically created
3. **Recruiters Apply** → Applications are received
4. **Accept Applications** → Recruiters are automatically added to chat room
5. **Communicate** → Can discuss candidates in the chat room

### For Organizations

1. **Create Organization** → Organization room is automatically created
2. **Organization Members** → Can access the organization chat room
3. **Internal Communication** → Team can communicate in the room

### Chat Interface Access

Users access chat through:
- **API Endpoints**: The platform provides REST API endpoints for:
  - Getting user's chat token
  - Listing user's rooms
  - Retrieving messages from a room
  - Sending messages to a room

**Frontend Integration**:
- The frontend application would call these APIs
- A chat UI would be built using these APIs
- Users would see chat rooms in their interface
- Messages would be displayed in real-time (with polling or WebSocket)

---

## Business Rules & Policies

### Automatic Room Creation Rules

**Job Rooms**:
- ✅ Created when job status = "open"
- ❌ NOT created for draft jobs
- ❌ NOT created for closed/on-hold jobs
- ✅ Created only once per job (not recreated if already exists)

**Organization Rooms**:
- ✅ Created when organization is created
- ✅ Created regardless of organization status
- ✅ Created only once per organization

### Access Control Rules

**Job Room Access**:
- Job creator: Always has access (from room creation)
- Recruiters: Only if their application status = "accepted"
- Access is automatically managed based on application status
- No manual invitation process

**Organization Room Access**:
- Currently public (anyone with link can join)
- Access should be managed through organization membership (platform feature)

### Membership Management Rules

**Adding Members**:
- Automatic: Based on application status changes
- Immediate: No approval or invitation process
- Silent: Users are added without notification (they discover the room when they check)

**Removing Members**:
- Automatic: Based on application status changes
- Immediate: Access is revoked immediately
- Silent: Users lose access without notification

### Message Rules

**Who Can Message**:
- Any user with room access can send messages
- No moderation or approval required
- Messages are sent immediately

**Message Visibility**:
- All room members can see all messages
- New members see full history
- No private messages within rooms

---

## Troubleshooting & Support

### Common Issues

#### Users Can't Access Chat

**Possible Causes**:
1. Matrix server is not configured or unavailable
2. User's chat account creation failed
3. User's access token is missing or invalid

**Resolution**:
- Check Matrix server configuration
- Verify Matrix server is running
- Check user's `matrix_user_id` and `matrix_access_token` in database
- User may need to re-register (chat account will be recreated)

#### Rooms Not Created

**Possible Causes**:
1. Matrix server is unavailable
2. Job creator doesn't have a chat account
3. Room creation failed silently

**Resolution**:
- Check if job has `matrix_room_id` in database
- Verify job creator has chat account
- Check Matrix server logs
- Room can be manually created if needed

#### Users Not Added to Rooms

**Possible Causes**:
1. User doesn't have a chat account
2. Application status change didn't trigger room addition
3. Room addition failed silently

**Resolution**:
- Verify user has `matrix_user_id` in database
- Check application status is "accepted"
- Verify job has `matrix_room_id`
- User can be manually added to room if needed

#### Messages Not Sending

**Possible Causes**:
1. User's access token is invalid or expired
2. User doesn't have room access
3. Matrix server is unavailable

**Resolution**:
- Verify user has valid access token
- Check user has room access
- Verify Matrix server is running
- May need to regenerate user's access token

### Support Scenarios

**User Reports Missing Chat Account**:
- Check if user has `matrix_user_id` in database
- If missing, chat account creation likely failed
- Can manually trigger account creation
- User may need to re-register

**User Reports Missing Room**:
- Check if job/organization has `matrix_room_id` in database
- If missing, room creation likely failed
- Can manually create room
- May need to republish job or recreate organization

**User Reports Access Issues**:
- Verify user's application status (for job rooms)
- Check room membership on Matrix server
- Verify user has valid access token
- May need to re-add user to room

### Monitoring & Health Checks

**Key Metrics to Monitor**:
1. **User Account Creation Success Rate**: How many users successfully get chat accounts
2. **Room Creation Success Rate**: How many jobs/orgs successfully get rooms
3. **Room Membership Accuracy**: Are users correctly added/removed based on status
4. **Message Delivery Success Rate**: Are messages being sent successfully
5. **Matrix Server Availability**: Is the Matrix server responding

**Health Check Indicators**:
- Matrix server is reachable
- Admin token is valid
- Room creation is working
- User account creation is working

---

## Future Considerations

### Potential Enhancements

1. **Room Closure**: Automatically close/archive rooms when jobs are closed
2. **Notification System**: Notify users when they're added to rooms
3. **Rich Messaging**: Support for file attachments, images, etc.
4. **Message Moderation**: Ability to moderate or delete messages
5. **Private Messages**: Direct messaging between users
6. **Room Roles**: Different permission levels (admin, member, read-only)
7. **Message Search**: Search functionality within rooms
8. **Room Analytics**: Track message activity, engagement metrics
9. **Retention Policies**: Automatic message deletion after X days
10. **Room Templates**: Pre-configured room settings for different job types

### Operational Considerations

1. **Backup Strategy**: How to backup Matrix server data
2. **Disaster Recovery**: Recovery procedures if Matrix server fails
3. **Scaling**: How to handle increased load as user base grows
4. **Migration**: How to migrate to a new Matrix server if needed
5. **Compliance**: Ensure chat data meets regulatory requirements (GDPR, etc.)

---

## Summary

The Matrix chat integration provides seamless, automatic chat functionality for the platform:

- **Every user** gets a chat account automatically
- **Every published job** gets a dedicated chat room
- **Every organization** gets a dedicated chat room
- **Access is automatic** based on application status
- **Messages are secure** and stored on Matrix server
- **History is preserved** for all room members
- **System is resilient** - chat failures don't break the platform

The integration is designed to "just work" - users don't need to think about it, and it enhances collaboration without adding complexity to the user experience.
