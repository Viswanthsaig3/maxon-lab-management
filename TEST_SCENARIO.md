# Test Scenario for Patient Capacity Display

## Setup in Capacity Management

### Employee Types:
- **Doctor**: 160 patients/day
- **Lab Technician**: 150 patients/day  
- **Nurse**: 200 patients/day

### Services:
- **Blood Test**: 1 Lab Technician
- **X-Ray**: 1 Doctor + 1 Nurse
- **ECG**: 1 Nurse

## Project Planning Test

### Input:
- Project Name: "Company Health Checkup"
- Total Employees: 1200
- Working Days: 5
- Selected Tests: Blood Test, X-Ray, ECG

### Expected Output:

#### Service Configuration Details:
| Service | Category | Employee Requirements | Participants |
|---------|----------|----------------------|-------------|
| Blood Test | diagnostic | 1 × Lab Technician (150 patients/day each) | 1200 |
| X-Ray | diagnostic | 1 × Doctor (160 patients/day each)<br>1 × Nurse (200 patients/day each) | 1200 |
| ECG | diagnostic | 1 × Nurse (200 patients/day each) | 1200 |

#### Staff Requirements:
| Staff Type | Capacity/Day per Person | Max Required | Available | Status |
|------------|------------------------|-------------|-----------|--------|
| Doctor | **160** patients/day | 2 | 0 | Need 2 more |
| Lab Technician | **150** patients/day | 2 | 0 | Need 2 more |
| Nurse | **200** patients/day | 2 | 0 | Need 2 more |

#### Daily Employee Distribution:
| Day | Doctor | Lab Technician | Nurse | Total Capacity | Participants | Workload % |
|-----|--------|---------------|--------|----------------|-------------|------------|
| Day 1 | 2 | 2 | 2 | 1020 | 720 | 71% |
| Day 2 | 2 | 2 | 2 | 1020 | 480 | 47% |
| Day 3 | 0 | 0 | 0 | 0 | 0 | 0% |
| Day 4 | 0 | 0 | 0 | 0 | 0 | 0% |
| Day 5 | 0 | 0 | 0 | 0 | 0 | 0% |

This shows:
1. **Patient capacity from services** - Each employee type shows their daily capacity
2. **Service requirements** - Shows what each service needs
3. **Optimized distribution** - Completes work in 2 days instead of spreading over 5
4. **Clear capacity visibility** - Shows exactly how many patients each employee can handle