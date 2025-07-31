# Page History Table Requirements

## Core Concept
Track mode and status transitions for items by creating historical records from revision data, with support for multiple hafizes and advanced graduation logic.

## Data Flow Overview
```
Initial State (hafiz_id, mode=NULL, status=6) 
    ↓
Process Revisions Chronologically (by hafiz-item combination)
    ↓  
Generate Transition Records (ONLY on mode changes)
    ↓
Apply Null Rules & Advanced Graduation Logic
    ↓
Store in page_history Table
```

## Key Logic Rules

### Multi-Hafiz Support
- Process each **hafiz_id + item_id combination** separately
- Each hafiz can have independent progression for the same item
- All filtering and state tracking is done per hafiz-item pair

### Mode Change Detection
- **Only create records when mode_id changes** from previous state
- Skip revisions that don't change the mode
- Track transitions: NULL→2, 2→3, 3→4, etc.

### Status Derivation
- **Mode 1** → Status 1
- **Mode 2,3,4** → Status 4  
- **Mode 5** → Status 5

### Null Value Rules (Applied in create_transition_record)
- **Same Status**: If `from_status == to_status`, set both to `NULL`
- **Same Mode**: If `from_mode == to_mode`, set both to `NULL`

### Date Logic for Mode 5 Transitions
- **Default**: Use `revision_date` from revision record
- **Mode 5 Special Case**: If transitioning **TO mode 5** AND `srs_start_date` exists in `hafizs_items`, use `srs_start_date`
- **Simplified Logic**: No longer checks if hafizs_items mode matches - just checks if srs_start_date is not null

### Advanced Graduation Logic (Priority Order)

#### Rule 1: Initial State (Highest Priority)
- **If `from_status == 6`** → `graduated_by = 'user'`
- Calculate `reps` if transitioning from mode 3,4,5: "count/denominator"

#### Rule 2: From Mode 2
- **If `from_mode == 2`** → `graduated_by = 'system'`, `reps = NULL`

#### Rule 3: To Mode 5 - Rating Check
- **If `to_mode == 5`** → Check previous two revision ratings
- If both previous ratings are `-1` (bad) → `graduated_by = 'system'`
- Otherwise → `graduated_by = 'user'`
- `reps = NULL` for mode 5 transitions

#### Rule 4: Standard Logic (Lowest Priority)
- **Only applies if from_mode in [3,4,5]** and none of above rules apply
- Count revisions under the `from_mode` for that hafiz-item
- **Denominators**: Mode 3,4 = 7, Mode 5 = 11
- **Format**: "count/denominator" (e.g., "3/7")
- **Graduation**: `system` if count ≥ denominator, else `user`

### Final State Reconciliation
- After processing all revisions, compare final state with `hafizs_items`
- If different, create additional transition record
- Apply same date logic and graduation rules
- Use `hafiz_id + item_id` to match records in `hafizs_items`

## Processing Algorithm

1. **Group by hafiz_id + item_id combinations**
2. **For each combination:**
   - Start with `current_mode = NULL, current_status = 6`
   - Sort revisions by `revision_date` ASC
   - Process each revision with index tracking
3. **For each revision:**
   - Calculate `new_mode` and `new_status` using mapping
   - **Only proceed if `current_mode != new_mode`**
   - Apply date logic (srs_start_date vs revision_date)
   - Apply graduation rules with revision index for rating checks
   - Create transition record with null value rules
   - Update current state
4. **Final reconciliation:**
   - Compare with hafizs_items current state
   - Create final transition if different
   - Apply same logic without revision index

## Result Schema
```sql
CREATE TABLE page_history (
    hafiz_id INT,
    item_id INT,
    pages INT,          -- Debug field from hafizs_items.page_number
    date DATETIME,
    from_status INT,    -- NULL if same as to_status
    to_status INT,      -- NULL if same as from_status  
    from_mode INT,      -- NULL if same as to_mode
    to_mode INT,        -- NULL if same as from_mode
    reps VARCHAR(10),   -- Format: "numerator/denominator" or NULL
    graduated_by ENUM('system', 'user', NULL)
);
```

## Key Implementation Details

### Revision Index Tracking
- Required for Rule 3 (to_mode = 5) to check previous ratings
- Use `enumerate()` when processing revisions to get current index
- Look back 2 positions: `item_revisions.iloc[current_revision_index-2:current_revision_index]`

### Date Field Population
- Default: `revision_date` from current revision
- Mode 5 override: `srs_start_date` from hafizs_items if not null
- Final reconciliation: Uses last revision date or srs_start_date

### Pages Field (Debug)
- Temporary field populated from `hafizs_items.page_number`
- Lookup based on `item_id` match

## Example Output
Each row represents one **mode transition** with:
- Proper hafiz isolation
- Null values where from/to are identical  
- Context-aware graduation logic
- Accurate date assignment for SRS transitions