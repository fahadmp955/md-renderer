# Pack Opening Process Analysis

This document outlines the detailed steps involved in the pack opening flow on SLABX and provides an analysis of the `NO_INVENTORY_AVAILABLE_FOR_BUCKET_floor` exception.

## Pack Opening Workflow

The pack opening process is orchestrated by the `OpenPackUseCase` and involves multiple coordinated services.

### 1. Pre-Opening Validation
*   **Availability Check**: Verifies the pack exists and is in an `active` state.
*   **Quantity Constraints**: Checks that the requested quantity is within the `minQty` and `maxQty` limits defined in the pack's purchase rules.
*   **Cost Calculation**: Calculates the total price based on `priceUsd` and requested quantity.

### 2. Transaction Initiation
*   **Purchase Record**: Creates a `PackPurchase` record with status `PENDING`.
*   **Funds Hold**: Calls the Wallet Service to place a "hold" on the user's balance. This ensures funds are reserved but not yet fully debited until the items are successfully generated.

### 3. Supply Management
*   **Supply Decrement**: Decrements the `total_supply` of the pack in the database to prevent over-selling.

### 4. Rip Execution (Iterative per Pack)
For each pack opened in the bundle, `executeSingleRip` is called:

#### A. RNG Draw
*   The `RngService` fetches the `OddsConfig` for the pack.
*   A random float between `0.0` and `1.0` is generated.
*   The system iterates through the `OddsItems` (buckets) and calculates cumulative probabilities.
*   **Selection**: The first bucket where the random value is less than or equal to the cumulative probability is selected (e.g., `floor`).
*   **Outcome**: An `RngDraw` record is created with the selected `label`.

#### B. Inventory Allocation (The Failure Point)
*   The `InventoryService` is called with the `packId` and the `bucket` label from the RNG draw.
*   It searches the `inventory_allocations` table for a record where:
    *   `pack_id` matches the current pack.
    *   `status` is `active`.
    *   `allocation_group` matches the RNG label (e.g., `floor`).
*   **Exception**: If no matching record is found, the `NO_INVENTORY_AVAILABLE_FOR_BUCKET_floor` error is thrown.

#### C. Slab Generation
*   If inventory is found, it is marked as `CONSUMED`.
*   A new `Slab` record is created for the user, inheriting metadata (display name, grade, image URL) from the consumed inventory item.

### 5. Finalization
*   **Wallet Debit**: The reserved hold is consumed and a permanent debit is recorded.
*   **Purchase Completion**: The `PackPurchase` status is updated to `COMPLETED`.
*   **Activity Logging**: The action is recorded in the user's activity feed.
*   **Reward Points**: Loyalty points are awarded based on the pack's value.

---

## Root Cause Analysis: `NO_INVENTORY_AVAILABLE_FOR_BUCKET_floor`

The error occurs during **Step 4.B** (Inventory Allocation).

### Potential Causes

1.  **Inventory/Odds Mismatch**:
    *   The `odds_items` table contains a bucket labeled `floor` with a non-zero probability.
    *   However, no items have been uploaded/allocated to this pack with the `allocation_group` of `floor`.
    *   **Result**: The RNG selects `floor`, but the inventory system has nothing to give.

2.  **Inventory Exhaustion**:
    *   Inventory for the `floor` bucket was originally present but has been completely consumed by previous openings.
    *   **Result**: The pack remains "Active" because it has total supply, but a specific sub-pool (bucket) is empty.

3.  **Label Casing/Typo**:
    *   The `OddsItem` label is `floor` (lowercase), but the `InventoryAllocation` group is `Floor` (capitalized) or vice versa.
    *   **Result**: Exact string matching in the repository fails.

4.  **Inactive Allocations**:
    *   Inventory items exist for the `floor` bucket, but their status is not `active` (e.g., they are still in `reserved` or `consumed` state).

### Recommended Verification Steps

*   **Check Database Counts**:
    ```sql
    SELECT allocation_group, status, COUNT(*) 
    FROM inventory_allocations 
    WHERE pack_id = '[YOUR_PACK_ID]' 
    GROUP BY allocation_group, status;
    ```
*   **Verify Odds Configuration**:
    ```sql
    SELECT label, probability 
    FROM odds_items 
    WHERE odds_config_id = (SELECT id FROM odds_configs WHERE pack_id = '[YOUR_PACK_ID]' AND is_active = true);
    ```
*   **Compare Labels**: Ensure the strings in `odds_items.label` exactly match `inventory_allocations.allocation_group`.
