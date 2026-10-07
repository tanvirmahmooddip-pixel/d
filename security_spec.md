# Firebase Security Specification & TDD Verification

## 1. Data Invariants
1. **Default-Deny Catch-All**: Any unspecified collections or paths are blocked unconditionally (`allow read, write: if false;`).
2. **Identity Integrity**: All mutations require authenticated staff access (`isSignedIn()`).
3. **ID Poisoning Protection**: Path variables must conform to `isValidId()` regex and maximum length limits.
4. **Structural & Field Boundaries**:
   - `customers`: Name, phone, and credit limit cannot exceed maximum defined sizes.
   - `products`: Product prices and stock counts must be valid numbers; names bounded by 150 characters.
   - `sales`: Total calculations and line items cannot contain arbitrary ghost keys.
   - `repairTickets`: Job cards must have bounded strings and valid statuses.
   - `branchTransfers`: Must transition through valid statuses (`draft`, `in_transit`, `received`, `cancelled`).
   - `pendingServices`: Reason categories and descriptions must be bounded.
5. **No Blind Blanket Reads**: Read operations require authenticated ERP users.

## 2. The "Dirty Dozen" Malicious Payloads
1. **Payload 1 (Ghost Field Injection on Customer)**: `{ id: "cust-1", name: "Ali", phone: "+96891234567", currentCredit: 0, __isRootAdmin: true }` -> Rejected by `hasOnly` schema validation.
2. **Payload 2 (Unauthenticated Sale Write)**: Anonymous / unauthenticated request creating sale invoice -> Rejected by `request.auth != null`.
3. **Payload 3 (ID Poisoning Attack)**: Writing to `/customers/` with 2KB string ID -> Rejected by `isValidId(customerId)`.
4. **Payload 4 (Negative Stock Injection)**: Product payload with string for stock `"100"` instead of number -> Rejected by `data.stock is number`.
5. **Payload 5 (Oversized Description Flooding)**: Repair ticket with 50KB description string -> Rejected by `.size() <= 1000`.
6. **Payload 6 (Invalid Transfer State Shortcutting)**: Branch transfer with status `"hacked_bypass"` -> Rejected by status enum check.
7. **Payload 7 (Denial of Wallet Junk Array)**: Sale transaction with 500 bogus nested elements -> Rejected by `.size() <= 100`.
8. **Payload 8 (Arbitrary System Config Overwrite)**: Writing unauthorized keys into `/systemConfig/company` -> Rejected by schema validator.
9. **Payload 9 (Customer Phone Junk Injection)**: Customer phone string with 500 characters -> Rejected by `data.phone.size() <= 32`.
10. **Payload 10 (Zero-Price Spoofing on Product)**: Setting costPrice to non-number -> Rejected by `data.costPrice is number`.
11. **Payload 11 (Unauthenticated Pending Service Status Alteration)**: Unauthenticated user resolving repair bottleneck -> Rejected.
12. **Payload 12 (Unauthorized Root Collection Traversal)**: Accessing `/unknown_collection/secrets` -> Rejected by default deny catch-all.
