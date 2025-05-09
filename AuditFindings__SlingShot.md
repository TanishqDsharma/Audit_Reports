### Medium

#### [M-1]

**Description:**



**Impact:**

**POC**

**Mitigation:**

### Non-Critical

### Gas

#### [G-1] `ConcatStrings::prependNumber` function was not used in entire codebase

**Description:** Unused fuctions should be removed from the codebase to avoid payying for unnecessary gas spent on deployment. 

#### [G-2] Avoid using `block.timestamp` for deadlines

**Description:** Use a contant value for deadline instead of `block.timestamp` as EVM need to perform a state read operation to get the vaule of `block.timestamp`

#### [G-3]
