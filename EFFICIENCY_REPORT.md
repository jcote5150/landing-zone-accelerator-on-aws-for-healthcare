# Code Efficiency Report

This report identifies several areas in the codebase where efficiency improvements can be made.

## 1. Double Array Iteration in ec2-instance-profile-permissions/index.js (Lines 75-76)

**Location:** `source/custom-config-rules/ec2-instance-profile-permissions/index.js`

**Issue:** The code iterates over the same array twice using separate `.map()` calls to extract different properties:

```javascript
const existingPolicyNames = configurationItem.configuration.attachedManagedPolicies.map(p => p.policyName);
const existingPolicyArns = configurationItem.configuration.attachedManagedPolicies.map(p => p.policyArn);
```

**Impact:** This results in O(2n) time complexity when O(n) would suffice.

**Recommendation:** Combine into a single iteration using `reduce()` or a single loop:

```javascript
const { existingPolicyNames, existingPolicyArns } = configurationItem.configuration.attachedManagedPolicies.reduce(
  (acc, p) => {
    acc.existingPolicyNames.push(p.policyName);
    acc.existingPolicyArns.push(p.policyArn);
    return acc;
  },
  { existingPolicyNames: [], existingPolicyArns: [] }
);
```

## 2. Redundant Condition Checks in attach-ec2-instance-profile/index.js (Lines 54-66)

**Location:** `source/custom-config-rules/attach-ec2-instance-profile/index.js`

**Issue:** The code checks `configurationItem.configuration` twice in consecutive conditions:

```javascript
if (configurationItem.configuration && configurationItem.configuration.iamInstanceProfile) {
  // ...
} else if (configurationItem.configuration && !configurationItem.configuration.iamInstanceProfile) {
  // ...
} else {
  // TODO retrieve from api call
}
```

**Impact:** The second condition redundantly checks `configurationItem.configuration` when it's already known to be truthy (since the first condition failed but included the same check).

**Recommendation:** Restructure the conditions:

```javascript
if (!configurationItem.configuration) {
  // Handle missing configuration
} else if (configurationItem.configuration.iamInstanceProfile) {
  // COMPLIANT
} else {
  // NON_COMPLIANT
}
```

## 3. Redundant Condition Checks in ec2-instance-profile-permissions/index.js (Lines 64-74)

**Location:** `source/custom-config-rules/ec2-instance-profile-permissions/index.js`

**Issue:** Similar pattern of redundant condition checking:

```javascript
if (configurationItem.configuration && !configurationItem.configuration.instanceProfileList) {
  // ...
} else if (configurationItem.configuration && configurationItem.configuration.instanceProfileList.length === 0) {
  // ...
} else if (configurationItem.configuration) {
  // ...
}
```

**Impact:** Multiple redundant checks of `configurationItem.configuration`.

**Recommendation:** Check `configurationItem.configuration` once at the beginning and restructure the nested conditions.

## 4. Unreachable Code Paths

**Locations:**
- `source/custom-config-rules/attach-ec2-instance-profile/index.js` (Lines 65-71)
- `source/custom-config-rules/ec2-instance-profile-permissions/index.js` (Lines 109-116)

**Issue:** Both files contain unreachable code after exhaustive condition checks, with TODO comments and incorrect annotations:

```javascript
} else {
  // TODO retrieve from api call
}

return {
  complianceType: 'NON_COMPLIANT',
  annotation: 'The resource logging destination is incorrect',
};
```

**Impact:** Dead code that can never execute, with misleading error messages about "logging destination" in files that check IAM profiles.

**Recommendation:** Either implement the TODO or remove the unreachable code and fix the annotation.

## 5. Duplicate Configuration Files

**Issue:** The repository contains duplicate configuration files in both the root directory and the `config/` subdirectory:

| Root Directory | config/ Directory |
|----------------|-------------------|
| `service-control-policies/scp-accelerator1.json` | `config/service-control-policies/scp-accelerator1.json` |
| `service-control-policies/scp-accelerator2.json` | `config/service-control-policies/scp-accelerator2.json` |
| `iam-policies/boundary-policy.json` | `config/iam-policies/boundary-policy.json` |
| `bucket-policies/central-log-bucket.json` | `config/bucket-policies/central-log-bucket.json` |
| And many more... | |

**Impact:** Maintenance overhead, risk of configuration drift, and confusion about which files are authoritative.

**Recommendation:** Consolidate to a single location and use symbolic links or references if needed.

## Summary

| Issue | Severity | Effort to Fix |
|-------|----------|---------------|
| Double array iteration | Medium | Low |
| Redundant condition checks (file 1) | Low | Low |
| Redundant condition checks (file 2) | Low | Low |
| Unreachable code paths | Low | Low |
| Duplicate configuration files | Medium | Medium |

## Selected Fix

For this PR, we will fix **Issue #1: Double Array Iteration** as it provides a clear performance improvement with minimal risk.
