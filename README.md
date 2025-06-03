# Gradebook


### Scheme

```mermaid
graph TD

    %% Entry Layer
    Client[Client Apps Web / Mobile]
    Gateway["API Gateway / BFF
    Handles auth, routing, rate-limits"]

    %% Core Services
    IAM["Identity Service
    - /auth/login
    - /auth/register
    - /auth/refresh
    Roles: STUDENT, TEACHER
    Token: JWT
    Payload: {email, password}"]

    UserProfileService["User Profile Service
    - /users/:id
    - /users/:id/avatar
    Stores metadata
    Payload: {name, email, role}"]

    SubjectService["Subject Service
    - /subjects
    - /subjects/:id
    - /subjects/:id/teachers
    Stores subject info
    Payload: {title, credits, semester}"]

    EnrollmentService["Enrollment Service
    - /enrollments
    - /enrollments/:id
    - /students/:id/subjects
    Links students to subjects
    Payload: {studentId, subjectId}"]

    GradeService["Grade Service
    - /grades
    - /grades/:id
    - /students/:id/grades
    Assign/view grades
    Payload: {studentId, subjectId, grade, teacherId}"]

    NotificationService["Notification Service
    - /notifications
    - Event-based
    Sends grade alerts
    Payload: {type, recipientId, message}"]

    AuditService["Audit Logging Service
    - /audit
    - internal only
    Logs sensitive ops
    Payload: {actorId, action, timestamp}"]

    %% Databases
    IAMDB[(IAM DB)]
    UserProfileDB[(User DB)]
    SubjectDB[(Subject DB)]
    EnrollmentDB[(Enrollment DB)]
    GradeDB[(Grade DB)]
    NotificationDB[(Notification DB)]
    AuditDB[(Audit Log DB)]

    %% Infra
    EventBus["Kafka / RabbitMQ
    Async events: GradeAssigned, EnrollmentCreated"]

    RedisCache["Redis / In-Memory Cache
    Used for fast access, token validation"]

    Prometheus[Prometheus Metrics Exporter]
    Grafana["Grafana Dashboard
    Visualizes Prometheus stats"]
    ELKStack["ELK / Loki
    Central log aggregation"]

    %% Client routes through Gateway
    Client --> Gateway

    %% Gateway routes
    Gateway --> IAM
    Gateway --> UserProfileService
    Gateway --> SubjectService
    Gateway --> EnrollmentService
    Gateway --> GradeService
    Gateway --> NotificationService

    %% Services and DBs
    IAM --> IAMDB
    UserProfileService --> UserProfileDB
    SubjectService --> SubjectDB
    EnrollmentService --> EnrollmentDB
    GradeService --> GradeDB
    NotificationService --> NotificationDB
    AuditService --> AuditDB

    %% Event communication
    GradeService -- "emit GradeAssigned" --> EventBus
    EnrollmentService -- "emit EnrollmentCreated" --> EventBus
    IAM -- "UserCreated" --> EventBus
    EventBus --> NotificationService
    EventBus --> AuditService

    %% Caching
    GradeService --> RedisCache
    SubjectService --> RedisCache
    EnrollmentService --> RedisCache

    %% Observability
    IAM --> Prometheus
    SubjectService --> Prometheus
    GradeService --> Prometheus
    EnrollmentService --> Prometheus
    NotificationService --> Prometheus
    Gateway --> Prometheus
    Gateway --> ELKStack
    AuditService --> ELKStack
```




# Gradebook Microservices System - Requirements

This document outlines the functional requirements for the Gradebook microservices system, expressed as user stories. These user stories capture the system's features from the perspective of different users (Students, Teachers, and Administrators).

## Table of Contents
1.  [User Roles](#1-user-roles)
2.  [User Management](#2-user-management)
3.  [Subject Management](#3-subject-management)
4.  [Enrollment Management](#4-enrollment-management)
5.  [Grade Management](#5-grade-management)
6.  [Notifications](#6-notifications)
7.  [Audit Logging](#7-audit-logging)

---

## 1. User Roles

*   **STUDENT:** A user who enrolls in subjects and receives grades.
*   **TEACHER:** A user who teaches subjects, manages enrollments, and assigns/manages grades.
*   **ADMINISTRATOR:** An internal user responsible for system oversight, user management, and audit log review (implicit from Audit Service).

## 2. User Management

### 2.1. User Registration & Authentication

*   **As a new user,** I want to be able to register with my email and a secure password, so that I can access the Gradebook system.
*   **As a registered user,** I want to be able to log in with my credentials, so that I can access my personalized Gradebook features.
*   **As an authenticated user,** I want my session to remain active for a reasonable period, and to be able to refresh my authentication token, so that I don't have to log in repeatedly.
*   **As an administrator,** I want to be able to assign or modify user roles (STUDENT, TEACHER) so that users have appropriate permissions.

### 2.2. User Profile Management

*   **As an authenticated user,** I want to view my profile details (name, email, role), so that I can confirm my information.
*   **As an authenticated user,** I want to be able to update my name and other non-sensitive profile information, so that my details are current.
*   **As an authenticated user,** I want to be able to upload and change my profile avatar, so that I can personalize my account.

## 3. Subject Management

*   **As a teacher,** I want to be able to create new subjects, specifying their title, credits, and semester, so that I can organize my courses.
*   **As a teacher,** I want to be able to view and manage the list of subjects I teach, so that I can keep track of my courses.
*   **As a teacher,** I want to be able to update the details of a subject I teach, so that I can correct or modify course information.
*   **As a student,** I want to be able to browse all available subjects, so that I can choose courses to enroll in.
*   **As a student,** I want to be able to view the details of a specific subject, so that I can understand its content and requirements.
*   **As a teacher,** I want to be able to assign other teachers to a subject, so that multiple instructors can manage a single course.

## 4. Enrollment Management

*   **As a student,** I want to be able to enroll myself in an available subject, so that I can officially join a course.
*   **As a student,** I want to be able to view all the subjects I am currently enrolled in, so that I can keep track of my academic commitments.
*   **As a student,** I want to be able to un-enroll from a subject, so that I can drop a course if needed.
*   **As a teacher,** I want to be able to view a list of all students enrolled in a subject I teach, so that I can manage my class roster.

## 5. Grade Management

*   **As a teacher,** I want to be able to assign a grade to a specific student for a specific subject I teach, so that I can record their academic performance.
*   **As a teacher,** I want to be able to update an existing grade for a student in a subject I teach, so that I can correct errors or reflect new assessments.
*   **As a teacher,** I want to be able to view all the grades for students in the subjects I teach, so that I have a comprehensive overview of their progress.
*   **As a student,** I want to be able to view all my assigned grades across all the subjects I am enrolled in, so that I can monitor my academic standing.
*   **As a teacher,** I want to be able to delete a grade for a student in a subject I teach, so that I can remove incorrect or irrelevant entries.

## 6. Notifications

*   **As a student,** I want to receive an automatic notification when a new grade is assigned or an existing grade is updated for me, so that I am immediately aware of changes to my academic record.
*   **As a student,** I want to receive an automatic notification upon successfully enrolling in a subject, so that I have confirmation of my registration.
*   **As a user,** I want to be able to view a history of all notifications sent to me, so that I can review past alerts.

## 7. Audit Logging

*   **As an administrator,** I want all sensitive actions (e.g., grade changes, user registrations, subject modifications) to be logged automatically, including who performed the action and when, so that I can ensure system integrity and accountability.
*   **As an administrator,** I want to be able to access and review these audit logs, so that I can investigate activities within the system.

---