# Capacity Management Redesign Summary

## Overview
The capacity management system has been redesigned to focus on **employee capacity** rather than service capacity. This change better reflects real-world resource planning where employees (doctors, nurses, etc.) handle a certain number of participants per day.

## Key Changes

### 1. Calculation Logic (`capacityService.ts`)
- **Old**: Calculated participants per day based on service time estimates
- **New**: Calculates based on employee daily capacity (e.g., doctor handles 160 patients/day)

#### New Algorithm Features:
- **Optimized Distribution**: Intelligently distributes employees across days to minimize total resources
- **Daily Workload Tracking**: Tracks workload percentage for each day
- **Peak Day Calculation**: Identifies days with highest employee requirements
- **Multiple Service Support**: Aggregates requirements across all selected services

#### Example:
- 1000 patients over 5 days with doctor capacity of 160/day
- Old: Would assign same number of doctors each day
- New: Optimizes to use 2 doctors for 3 days, 1 doctor for 2 days

### 2. Data Model Updates (`capacity.ts`)
- Added `participantsHandled` field to daily distribution tracking
- Enhanced calculation to support optimal resource allocation

### 3. UI Updates

#### Capacity Management Page
- **Old Display**: "Participants/Day" calculated from service time
- **New Display**: Shows employee requirements with individual capacities
  - Lists each employee type needed
  - Shows individual capacity per employee
  - Displays total service capacity

#### Project Planning Dashboard
- Added "Daily Employee Distribution" table showing:
  - Employee allocation for each day
  - Workload percentage per day
  - Optimized distribution across project duration

### 4. Benefits of New System

1. **Accurate Resource Planning**: Based on actual employee capacities
2. **Cost Optimization**: Minimizes total employee-days needed
3. **Better Utilization**: Shows workload percentages to identify under/over-utilization
4. **Flexibility**: Handles multiple services with different employee requirements
5. **Transparency**: Clear visibility into daily resource allocation

### 5. How It Works

1. **Define Employee Types**: Set daily capacity for each type (doctor: 160/day, nurse: 200/day, etc.)
2. **Configure Services**: Specify which employee types and how many are needed per service
3. **Plan Projects**: Select services and participant counts
4. **Get Optimized Plan**: System calculates optimal employee distribution across days

### 6. Testing
Created test scenarios in `capacityCalculation.test.ts` to verify:
- Single service optimization
- Multiple concurrent services
- Edge cases and workload balancing

## Next Steps
1. Add employee availability calendar integration
2. Implement cost optimization based on employee rates
3. Add visual charts for resource utilization
4. Enable manual override of automatic distribution