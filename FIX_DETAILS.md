# Property Revenue Dashboard - Bug Fix Details

This document provides a detailed explanation of the bugs identified in the Property Revenue Dashboard and the proposed implementation for fixing them.

## Technical Root Cause Analysis

### 1. Multi-tenancy Isolation (Privacy Concern)
**Issue**: Client B sometimes sees revenue numbers belonging to Client A.
**Root Causes**:
- **Cache Leak**: The Redis key was shared (`revenue:{property_id}`). Fixed by adding `tenant_id`.
- **Mock Fallback Leak**: In the `reservations.py` mock logic, if a tenant requested a property they didn't own, it defaulted to `tenant-a`'s data.
**Fix**: Added strict `tenant_id` checks in both Redis keys and mock data fallbacks to ensure $0.00 is returned if the property doesn't belong to the tenant.

### 2. Timezone-related Revenue Inaccuracy (March Totals)
**Issue**: Client A (Sunset Properties) March totals don't match internal records.
**Root Cause**: A reservation in the database is stored as `2024-02-29 23:30:00+00`. 
For `prop-001` (Europe/Paris timezone), this UTC time corresponds to `2024-03-01 00:30:00+01`.
The current backend logic performs a simple sum across all reservations based on UTC date filtering, which counts this reservation in February instead of March for Paris-based properties.

### 3. Revenue Precision Loss (Penny Differences)
**Issue**: Finance team noticed totals are "slightly off" by a few cents.
**Root Cause**: 
- **Backend**: In `dashboard.py`, the `Decimal` from the database is converted to a standard Python `float`. 
- **Frontend**: In `RevenueSummary.tsx`, `Math.round(data.total_revenue * 100) / 100` is used.
Binary floating-point arithmetic cannot accurately represent all decimals, leading to cumulative rounding errors in financial calculations.

## Proposed Changes

### Backend

#### [MODIFY] `backend/app/services/cache.py`
- Update `cache_key` generation to include `tenant_id` to ensure data isolation.
- `cache_key = f"revenue:{tenant_id}:{property_id}"`

#### [MODIFY] `backend/app/api/v1/dashboard.py`
- Keep revenue values as strings when transferring to the frontend to maintain high precision.

#### [MODIFY] `backend/app/services/reservations.py`
- Update `calculate_total_revenue` to:
    - Join with the `properties` table to retrieve property timezones.
    - Factor in timezones when calculating month-based revenue to ensure reservations are attributed to the correct month based on local time.
    - Implement strict `tenant_id` verification in mock data fallback to prevent cross-tenant leakage if DB is offline.

### Frontend

#### [MODIFY] `frontend/src/components/RevenueSummary.tsx`
- Implement proper decimal formatting for display without relying on `Math.round`.
- Ensure the frontend handles string-based revenue values from the API correctly.

## Verification Plan

### Manual Verification
1. **Privacy Fix**: Log in as different clients and verify that `prop-001` shows different, tenant-specific data without being served cached data from another client.
2. **Revenue Accuracy**: Confirm Client A's March total for `prop-001` includes the timezone-shifted reservation.
3. **Precision**: Verify that no "Precision Mismatch Detected" warnings appear.
