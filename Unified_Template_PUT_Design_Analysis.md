# Unified Template PUT Endpoint - Design Analysis & Implementation Plan

## Overview
This document provides a comprehensive analysis of implementing a unified PUT endpoint for templates, similar to the existing campaign unified PUT endpoint. The template system has additional complexities due to auto-generated naming, S3 file management, and LOB-dependent path structures.

## Current Template System Architecture

### Template Name Generation
- **Pattern**: `LOB_templateType_createDateTime`
- **Example**: `COMMERCIAL_EMAIL_2024-11-15T10-30-15`
- **Dependencies**: LOB field, templateType field, createDateTime

### S3 Path Structure
- **Base Pattern**: `s3://bucket/<LOB>/<status>/`
- **Staging**: `s3://csbd-email-utility-sitv1/COMMERCIAL/staging/`
- **Approved**: `s3://csbd-email-utility-sitv1/COMMERCIAL/approved/`
- **File Naming**: `<templateName>.html`

### Current Status Lifecycle
- `PENDINGAPPROVAL` → `APPROVED` → `CANCELED`
- No INACTIVE status (removed as per recent requirements)

## Critical Issues & Gaps Analysis

### 1. Template Name Dependency Chain ⚠️
**Issue**: Template name regeneration affects multiple system components

**Current Problems**:
- LOB change: `COMMERCIAL_EMAIL_2024-11-15T10-30-15` → `GBD_EMAIL_2024-11-15T10-30-15`
- TemplateType change: `COMMERCIAL_EMAIL_2024-11-15T10-30-15` → `COMMERCIAL_GENERIC_2024-11-15T10-30-15`
- Combined changes require complete name regeneration

**Impact Areas**:
- Database record templateName field
- S3 file naming
- Location field in database
- Audit trail consistency
- User recognition and traceability

### 2. S3 File Management Complexity ⚠️
**Issue**: Multi-dimensional file movement requirements

**Movement Scenarios**:
```
Scenario 1: LOB Change Only
OLD: s3://bucket/COMMERCIAL/approved/COMMERCIAL_EMAIL_2024-11-15T10-30-15.html
NEW: s3://bucket/GBD/staging/GBD_EMAIL_2024-11-15T10-30-15.html

Scenario 2: Status Change Only  
OLD: s3://bucket/COMMERCIAL/staging/COMMERCIAL_EMAIL_2024-11-15T10-30-15.html
NEW: s3://bucket/COMMERCIAL/approved/COMMERCIAL_EMAIL_2024-11-15T10-30-15.html

Scenario 3: Combined LOB + Status Change
OLD: s3://bucket/COMMERCIAL/approved/COMMERCIAL_EMAIL_2024-11-15T10-30-15.html  
NEW: s3://bucket/GBD/staging/GBD_EMAIL_2024-11-15T10-30-15.html

Scenario 4: TemplateType Change
OLD: s3://bucket/COMMERCIAL/approved/COMMERCIAL_EMAIL_2024-11-15T10-30-15.html
NEW: s3://bucket/COMMERCIAL/staging/COMMERCIAL_GENERIC_2024-11-15T10-30-15.html
```

**Complexity Factors**:
- Cross-LOB directory movements
- File renaming during moves
- Status-based folder changes
- Atomic operation requirements
- Rollback capabilities

### 3. Status Transition & Business Logic ⚠️
**Issue**: Status changes trigger cascading S3 operations

**Business Rules**:
- Any field update on APPROVED template → moves to PENDINGAPPROVAL
- PENDINGAPPROVAL → APPROVED requires approval workflow
- APPROVED templates must move from `/approved/` to `/staging/` on updates
- CANCELED templates are immutable

**S3 Implications**:
- Status demotion requires file movement from approved to staging
- Cross-LOB updates require both directory and status folder changes
- File availability during transition periods

### 4. Additional Critical Gaps Identified

#### 4.1 Transaction Consistency & Atomicity 🔴
**Gap**: No coordinated transaction management between DB and S3

**Problems**:
- DB update succeeds, S3 operation fails → Inconsistent state
- S3 operation succeeds, DB update fails → Orphaned files
- Partial updates during multi-step operations
- No rollback mechanism for failed operations

**Business Impact**:
- Data integrity violations
- Manual cleanup requirements
- System reliability issues
- User confusion from inconsistent states

#### 4.2 Concurrent Access & Race Conditions 🔴
**Gap**: No protection against simultaneous operations

**Scenarios**:
- User A updating template while User B approving same template
- Multiple users updating different fields simultaneously
- Approval process running while update is in progress
- S3 file conflicts during concurrent moves

**Risks**:
- Lost updates
- File corruption or conflicts
- Inconsistent approval states
- Database deadlocks

#### 4.3 File Versioning & History Management 🔴
**Gap**: No version control for template files

**Missing Capabilities**:
- No backup of previous template versions
- Cannot rollback to previous file content
- No audit trail of file changes
- Limited disaster recovery options

**Business Requirements**:
- Regulatory compliance may require version history
- Business users may need to revert changes
- Debugging requires historical file access
- Approval workflows may need comparison capabilities

#### 4.4 Error Handling & Recovery Mechanisms 🔴
**Gap**: Insufficient error recovery strategies

**Current Limitations**:
- Generic error messages for all failure types
- No automatic retry for transient failures
- No differentiation between recoverable/non-recoverable errors
- Missing user guidance for error resolution

**Required Improvements**:
- Specific error messages for different failure scenarios
- Automatic retry with exponential backoff
- Manual intervention workflows for critical failures
- User-friendly error explanations and next steps

#### 4.5 Performance & Resource Optimization 🟡
**Gap**: Inefficient operations for large files and high load

**Current Issues**:
- Synchronous S3 operations blocking user requests
- Large template files causing network bottlenecks
- No caching for frequently accessed templates
- Resource-intensive file copy operations

**Optimization Needs**:
- Asynchronous processing for large operations
- Progress tracking for long-running operations
- Caching strategies for template metadata
- Connection pooling for S3 operations

#### 4.6 Audit Trail & Compliance 🟡
**Gap**: Limited tracking and compliance capabilities

**Missing Features**:
- Detailed audit logs for template changes
- File-level change tracking in S3
- Approval workflow audit trail
- Compliance reporting capabilities

**Requirements**:
- Who changed what, when, and why
- File movement history in S3
- Approval decision audit trail
- Regulatory compliance reporting

#### 4.7 Validation & Integrity Checks 🟡
**Gap**: Limited validation of template and file integrity

**Missing Validations**:
- Template file format validation
- S3 file existence verification
- Template content integrity checks
- Business rule validation consistency

**Required Validations**:
- Pre-operation file existence checks
- Post-operation integrity verification
- Template content format validation
- Business rule compliance verification

## Comprehensive Solution Framework

### Phase 1: Foundation Services Architecture

#### 1.1 TemplateNameGenerator Service
```java
@Service
public class TemplateNameGenerator {
    // Generate template name based on LOB, type, and original createDateTime
    // Ensure consistency and uniqueness
    // Handle multiple LOB scenarios (array to underscore-separated)
}
```

**Responsibilities**:
- Generate consistent template names
- Handle LOB array to string conversion
- Maintain original createDateTime for consistency
- Validate name uniqueness

#### 1.2 S3PathManager Service
```java
@Service  
public class S3PathManager {
    // Construct S3 paths based on LOB and status
    // Handle path validation and existence checks
    // Generate source and target paths for moves
}
```

**Responsibilities**:
- Construct LOB-based S3 paths
- Validate path accessibility
- Generate move operation paths
- Handle path configuration management

#### 1.3 FileOperationOrchestrator Service
```java
@Service
public class FileOperationOrchestrator {
    // Coordinate complex file operations
    // Implement copy-then-delete pattern
    // Handle rollback scenarios
}
```

**Responsibilities**:
- Orchestrate multi-step file operations
- Implement atomic file moves
- Handle operation rollback
- Coordinate with transaction management

#### 1.4 TemplateTransactionManager Service
```java
@Service
public class TemplateTransactionManager {
    // Manage distributed transactions between DB and S3
    // Implement compensation patterns
    // Handle rollback and recovery
}
```

**Responsibilities**:
- Coordinate DB and S3 operations
- Implement saga pattern for distributed transactions
- Handle compensation actions
- Manage operation state tracking

### Phase 2: Core Business Logic Services

#### 2.1 TemplateUpdateOrchestrator Service
```java
@Service
public class TemplateUpdateOrchestrator {
    // Main coordination service for template updates
    // Determine update type and required actions
    // Orchestrate all dependent operations
}
```

**Key Methods**:
- `determineUpdateActions()` - Analyze what operations are needed
- `validateUpdatePermissions()` - Check user authorization
- `executeUpdate()` - Coordinate all update operations
- `handleUpdateFailure()` - Manage failure scenarios

#### 2.2 TemplateValidationEngine Service
```java
@Service
public class TemplateValidationEngine {
    // Comprehensive validation for template operations
    // Business rule validation
    // File integrity checks
}
```

**Validation Types**:
- Field-level validations
- Business rule compliance
- File existence and integrity
- User permission validation
- Concurrent operation detection

#### 2.3 TemplateVersionManager Service
```java
@Service
public class TemplateVersionManager {
    // Manage template versioning and history
    // Handle version snapshots
    // Provide rollback capabilities
}
```

**Features**:
- Version snapshot creation
- Historical version retrieval
- Rollback operation support
- Version comparison utilities

### Phase 3: Advanced Operational Services

#### 3.1 AsyncOperationManager Service
```java
@Service
public class AsyncOperationManager {
    // Handle long-running operations asynchronously
    // Provide operation status tracking
    // Send completion notifications
}
```

**Capabilities**:
- Background job execution
- Operation progress tracking
- Status notification system
- Queue management for high load

#### 3.2 TemplateAuditService Service
```java
@Service
public class TemplateAuditService {
    // Comprehensive audit logging
    // Compliance reporting
    // Change tracking
}
```

**Features**:
- Detailed change logging
- Audit report generation
- Compliance data collection
- Historical analysis capabilities

## Detailed Solution Strategies

### Strategy 1: Template Name Consistency Management

#### Approach: Preserve Original Timestamp
```
Principle: Use original createDateTime for all template name generations

Benefits:
- Maintains chronological consistency
- Ensures unique identification
- Preserves audit trail continuity
- Simplifies name collision handling

Implementation:
- Store original createDateTime separately
- Use for all name generation operations
- Never update this field for existing templates
- Handle timezone consistency
```

#### Name Generation Logic:
```java
public String generateTemplateName(List<String> lob, String templateType, LocalDateTime originalCreateDateTime) {
    String lobString = String.join("_", lob).toUpperCase();
    String formattedDateTime = originalCreateDateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-ddTHH-mm-ss"));
    return String.format("%s_%s_%s", lobString, templateType.toUpperCase(), formattedDateTime);
}
```

### Strategy 2: S3 File Movement Operations

#### Three-Phase Atomic File Operations

##### Phase A: Preparation & Validation
```java
public class FileOperationPlan {
    // Pre-flight validation
    - Verify source file exists
    - Check target location availability  
    - Validate permissions and quotas
    - Create operation transaction log
    - Lock resources for operation
}
```

##### Phase B: Copy & Verify Operations
```java
public class FileCopyOperation {
    // Safe copy operations
    - Copy file to target location with new name
    - Verify copy integrity (checksum validation)
    - Update database with new location (within transaction)
    - Log successful copy operation
    - Maintain source file until confirmation
}
```

##### Phase C: Cleanup & Finalization
```java
public class FileCleanupOperation {
    // Asynchronous cleanup
    - Delete source file (after confirmation)
    - Clean up temporary resources
    - Update operation logs
    - Send completion notifications
    - Update audit trail
}
```

#### Error Recovery Mechanisms:
```
Failure Scenarios & Recovery Actions:

1. Copy Operation Fails:
   → Keep original file intact
   → Rollback DB changes
   → Clean up partial copies
   → Return user-friendly error

2. DB Update Fails After Copy:
   → Delete copied file
   → Maintain original state
   → Log incident for investigation
   → Retry with exponential backoff

3. Cleanup Fails:
   → Log orphaned file details
   → Queue for background cleanup
   → Continue operation (success)
   → Alert administrators
```

### Strategy 3: Status Transition Matrix

#### Comprehensive State Management
```
Template Status Transitions & Required Actions:

PENDINGAPPROVAL → PENDINGAPPROVAL (Field Updates):
├── LOB Changed: Move between LOB directories in staging
├── TemplateType Changed: Rename file with new template name
├── Other Fields: No file operations needed
└── Actions: Update DB, regenerate name if needed, move file if needed

APPROVED → PENDINGAPPROVAL (Field Updates):
├── Any Field Change: Move from approved/ to staging/  
├── LOB Changed: Move between LOB directories + status change
├── TemplateType Changed: Move + rename with new template name
└── Actions: Update DB, change status, move file, regenerate name if needed

PENDINGAPPROVAL → APPROVED (Approval):
├── Keep current location and name
├── Move from staging/ to approved/
├── Update approval metadata
└── Actions: Update DB status, move file to approved, log approval

APPROVED → CANCELED (Cancellation):
├── No file operations needed
├── Update status only
├── Make record immutable
└── Actions: Update DB status, log cancellation, prevent further updates

CANCELED → * (Any Transition):
├── Block all operations
├── Return error message
├── Log attempted operation
└── Actions: Reject request, maintain current state
```

### Strategy 4: Transaction Management Architecture

#### Saga Pattern Implementation
```java
@Service
public class TemplateUpdateSaga {
    
    // Step 1: Validate and Prepare
    public SagaStep validateAndPrepare(TemplateUpdateRequest request) {
        // Validate all preconditions
        // Lock resources
        // Create compensation data
        // Return step result with rollback info
    }
    
    // Step 2: Execute Database Operations  
    public SagaStep executeDatabaseUpdate(Template template, SagaContext context) {
        // Update template record
        // Log changes
        // Prepare S3 operation data
        // Return step result with rollback info
    }
    
    // Step 3: Execute S3 Operations
    public SagaStep executeS3Operations(S3OperationPlan plan, SagaContext context) {
        // Execute file operations
        // Verify success
        // Update location in DB
        // Return step result with rollback info
    }
    
    // Step 4: Finalize and Cleanup
    public SagaStep finalizeOperation(SagaContext context) {
        // Clean up temporary resources
        // Send notifications
        // Update audit logs
        // Return final result
    }
    
    // Compensation Actions
    public void compensate(SagaStep failedStep, SagaContext context) {
        // Execute appropriate rollback actions
        // Clean up partial changes
        // Log failure details
        // Notify administrators if needed
    }
}
```

#### Transaction Coordination:
```java
@Transactional
public TemplateUpdateResult executeTemplateUpdate(TemplateUpdateRequest request) {
    SagaContext context = new SagaContext();
    
    try {
        // Execute saga steps
        SagaStep step1 = validateAndPrepare(request);
        context.addStep(step1);
        
        SagaStep step2 = executeDatabaseUpdate(template, context);
        context.addStep(step2);
        
        SagaStep step3 = executeS3Operations(s3Plan, context);
        context.addStep(step3);
        
        SagaStep step4 = finalizeOperation(context);
        context.addStep(step4);
        
        return TemplateUpdateResult.success(template);
        
    } catch (Exception e) {
        // Execute compensation for all completed steps
        compensateCompletedSteps(context);
        return TemplateUpdateResult.failure(e);
    }
}
```

### Strategy 5: Error Handling & Recovery

#### Error Classification System
```java
public enum ErrorCategory {
    VALIDATION_ERROR,    // 400-level: User input issues
    BUSINESS_RULE_ERROR, // 409-level: Business logic violations  
    RESOURCE_ERROR,      // 423-level: Resource conflicts/locks
    INFRASTRUCTURE_ERROR,// 500-level: System/network issues
    CRITICAL_ERROR      // 500-level: Requires immediate attention
}
```

#### Error Response Framework:
```java
@Component
public class TemplateErrorHandler {
    
    public ErrorResponse handleValidationError(ValidationException e) {
        return ErrorResponse.builder()
            .category(ErrorCategory.VALIDATION_ERROR)
            .message("Invalid input provided")
            .details(e.getFieldErrors())
            .suggestedActions(List.of("Review field requirements", "Check input format"))
            .retryable(false)
            .build();
    }
    
    public ErrorResponse handleInfrastructureError(S3Exception e) {
        return ErrorResponse.builder()
            .category(ErrorCategory.INFRASTRUCTURE_ERROR)  
            .message("S3 operation failed")
            .details(e.getErrorDetails())
            .suggestedActions(List.of("Retry operation", "Check network connectivity"))
            .retryable(true)
            .retryAfter(Duration.ofMinutes(1))
            .build();
    }
}
```

#### Recovery Strategies:
```
Error Type → Recovery Strategy

Validation Errors:
├── Return detailed field-level errors
├── Provide correction guidance
├── No system state changes
└── Log for analytics

Business Rule Violations:
├── Explain rule violation clearly
├── Suggest alternative workflows
├── Provide override options if applicable
└── Log for business review

Resource Conflicts:
├── Implement retry with backoff
├── Provide operation queuing
├── Estimate retry timing
└── Allow user cancellation

Infrastructure Failures:
├── Automatic retry (3 attempts)
├── Exponential backoff strategy
├── Circuit breaker pattern
├── Fallback to manual processing queue
└── Alert system administrators

Critical System Errors:
├── Immediate rollback of all changes
├── System health check initiation
├── Administrator alert escalation
├── User notification with incident ID
└── Manual intervention workflow
```

## Implementation Roadmap

### Phase 1: Foundation Infrastructure (Weeks 1-2)
**Deliverables**:
- [ ] TemplateNameGenerator Service
- [ ] S3PathManager Service  
- [ ] Basic error handling framework
- [ ] Unit tests for core services
- [ ] Integration test setup

**Success Criteria**:
- Template name generation works for all LOB scenarios
- S3 path construction handles all status transitions
- Error handling provides user-friendly messages
- All unit tests pass with >90% coverage

### Phase 2: Core Update Functionality (Weeks 3-4)
**Deliverables**:
- [ ] TemplateUpdateOrchestrator Service
- [ ] FileOperationOrchestrator Service
- [ ] TemplateTransactionManager Service
- [ ] Template update validation engine
- [ ] Basic S3 file operations

**Success Criteria**:
- Template field updates work correctly
- File operations handle all movement scenarios
- Transaction management prevents data inconsistency
- Rollback mechanisms function properly
- Integration tests pass for main workflows

### Phase 3: Advanced Features (Weeks 5-6)
**Deliverables**:
- [ ] AsyncOperationManager Service
- [ ] TemplateVersionManager Service
- [ ] TemplateAuditService Service
- [ ] Performance optimization features
- [ ] Comprehensive error handling

**Success Criteria**:
- Long-running operations execute asynchronously
- Version history tracks all changes
- Audit trail captures all operations
- Performance meets acceptable thresholds
- Error recovery works for all scenarios

### Phase 4: Production Hardening (Weeks 7-8)
**Deliverables**:
- [ ] Production monitoring setup
- [ ] Alerting and notification system
- [ ] Complete API documentation
- [ ] Load testing validation
- [ ] Security review completion

**Success Criteria**:
- System handles production load requirements
- Monitoring catches all critical issues
- Documentation enables easy integration
- Security review identifies no critical issues
- Go-live readiness confirmed

## Risk Assessment & Mitigation

### High-Risk Areas

#### 1. Data Integrity Risks 🔴
**Risk**: Inconsistent state between DB and S3 during operations
**Probability**: High | **Impact**: Critical

**Mitigation Strategies**:
- Implement comprehensive transaction management
- Use copy-before-delete pattern for all file operations
- Implement health checks to detect inconsistencies
- Create manual reconciliation procedures
- Maintain operation audit logs for troubleshooting

#### 2. Performance Degradation 🟡
**Risk**: Large file operations blocking user experience  
**Probability**: Medium | **Impact**: High

**Mitigation Strategies**:
- Implement asynchronous processing for large operations
- Add progress tracking and user notifications
- Use connection pooling for S3 operations
- Implement circuit breaker patterns
- Monitor and alert on performance thresholds

#### 3. File Loss or Corruption 🔴
**Risk**: Template files lost or corrupted during move operations
**Probability**: Low | **Impact**: Critical

**Mitigation Strategies**:
- Always copy before delete operations
- Implement checksum validation for file integrity
- Maintain backup copies during transitions
- Create file recovery procedures
- Implement automated backup processes

### Medium-Risk Areas

#### 4. Concurrent Operation Conflicts 🟡
**Risk**: Multiple users modifying same template simultaneously
**Probability**: Medium | **Impact**: Medium

**Mitigation Strategies**:
- Implement optimistic locking on template records
- Add operation conflict detection
- Provide clear error messages for conflicts
- Implement operation queuing for high contention
- Add user notification for operation status

#### 5. S3 Service Availability 🟡
**Risk**: AWS S3 service interruption affecting operations
**Probability**: Low | **Impact**: High

**Mitigation Strategies**:
- Implement retry with exponential backoff
- Add circuit breaker for S3 operations
- Queue operations during S3 outages
- Provide user status updates during outages
- Create manual recovery procedures

## Success Metrics & KPIs

### Functional Metrics
- **Template Update Success Rate**: >99.5%
- **Data Consistency Rate**: 100% (no DB/S3 mismatches)
- **Operation Rollback Success**: 100% for failed operations
- **Error Recovery Success**: >95% automatic recovery

### Performance Metrics  
- **Update Operation Response Time**: <2 seconds (excluding large file operations)
- **Async Operation Completion**: 95% within 5 minutes
- **S3 Operation Success Rate**: >99.9%
- **Database Transaction Success**: >99.9%

### User Experience Metrics
- **User Error Resolution**: <10 seconds average
- **Operation Status Visibility**: Real-time updates
- **Error Message Clarity**: 95% user satisfaction
- **Documentation Completeness**: 100% endpoint coverage

## Conclusion

The unified Template PUT endpoint requires a sophisticated architecture to handle the complexities of auto-generated naming, multi-dimensional S3 file operations, and status-based business logic. The proposed solution addresses all identified gaps while maintaining system reliability, performance, and user experience.

The phased implementation approach allows for iterative development and testing, reducing risk while ensuring comprehensive feature coverage. The emphasis on transaction management, error recovery, and audit capabilities ensures the solution meets enterprise-grade requirements for reliability and compliance.

Success depends on careful implementation of the transaction management layer, comprehensive testing of all error scenarios, and thorough performance validation under realistic load conditions.

---
*Document Version: 1.0*  
*Last Updated: November 17, 2025*  
*Document Owner: Development Team*
