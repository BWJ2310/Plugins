# @hydrooj/a11y Plugin

## Overview
The @hydrooj/a11y plugin is a performance testing and admin authorization utility for HydroOJ. Despite its name suggesting accessibility features, this plugin primarily focuses on system performance benchmarking and administrative functionality.

## Version
Current version: 0.0.2

## Features

### 1. Admin Authorization
- Automatically grants super admin privileges to user ID 2 upon registration
- Utilizes UserModel for admin status management

### 2. Performance Testing System
A comprehensive testing suite that evaluates various aspects of system performance:

#### Test Categories:
- Memory Operations
- Matrix Operations
- Random Number Generation
- Set Operations
- Array Operations
- Mathematical Computations

## Technical Implementation

### Imported Packages
The plugin utilizes several core packages from HydroOJ:

#### Core Components
- `Context`: Application context management
- `Schema`: Data validation and schema definitions
- `UserModel`: User operations management

#### Problem & Record Management
- `ProblemModel`: Problem creation and management
- `RecordModel`: Test records and submissions management
- `STATUS`: Submission status enumeration
- `yaml`: YAML file parsing/dumping
- `sleep`: Async delay utility

### Models Usage

#### ProblemModel Operations
```typescript
- ProblemModel.get(): Retrieves test problems
- ProblemModel.add(): Creates new test problems
- ProblemModel.addTestdata(): Adds test cases
