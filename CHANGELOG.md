## v2.0.0

## New Features:

1. Asset tag default is the serial number on create
    -When creating new assets, the script now sets asset_tag to the device serial number (fallbacks: deviceName → name → serialNumber) so Snipe-IT's asset_tag is always populated and stable.
    -Rationale: asset_tag is required by many Snipe-IT setups and often used as the canonical asset identifier in the UI/workflows.

2. Safe update semantics
    -When the script finds an existing asset by serial number, it will only update a limited set of attributes: name (device display name), assigned_user (owner), and status_id only if status change is appropriate (see rules below).
    -It will not modify asset_tag, serial, or other inventory fields by default. This prevents accidental overwrites of manual edits in Snipe-IT.

3. Flexible Snipe-IT user matching
    -New find_snipe_user() function tries multiple matches in order:
        -UserPrincipalName (UPN / full email)
        -emailAddress (if different)
        -Explicit userName (if present)
        -username derived from UPN (string before @)
        -displayName (contains match)
        -userId fallback
    -Returns a Snipe-IT user id (or None) and logs matching attempts when -v/--verbose is used.

4. Automatic checkout (traceable owner assignment)
    -If an asset is created with an assigned owner or an owner is newly assigned during update, the script will create a checkout record in Snipe-IT for that owner (so Snipe-IT shows the device as checked out to the user).
    -This mirrors a real-world workflow: assigning an owner = device is out-of-inventory and in use.

5. Status mapping and compliance handling
    -Script looks up Snipe-IT status labels by name: Pending, Ready to Deploy, Deployed (checked out to user).
    -New logic:
        -If device is noncompliant in Intune → set status to Pending.
        -If device is compliant and no owner → Ready to Deploy.
        -If device has owner and is compliant → assign owner + checkout 
        -If a requested status label is missing in Snipe-IT, the script will fall back to the first search result but logs that fallback — you should verify Snipe-IT labels match the expected names.

6. Idempotent update behavior
-The script detects no-op updates and skips API calls where there are no changes for the fields it manages. This reduces noise and avoids unnecessary API rate usage.

7. Resilient API response handling
    -Improved defensive code for varying Snipe-IT response shapes (rows wrappers, lists, ints). Avoids crashes caused by inconsistent API returns.

8. More robust CLI and testing options
    -Existing flags kept and respected:
        - --dry-run, -v/--verbose, --do-not-verify-ssl, --slowdown, --suppress-tls-warning
    -Use --dry-run -v for a full, safe preview.


## Bug fixes:

1. Missing asset_tag on create
    -v1.0.0 created assets without asset_tag, causing Snipe-IT validation errors. v2.0.0 assigns serialNumber as asset_tag by default.

2. Unconditional status overwrites
    -v1.0.0 often set the asset status_id for existing assets unconditionally, causing previously-manual statuses to be overwritten. v2.0.0 avoids changing existing statuses except where logically required (e.g., new owner becomes assigned → checkout).

3. Owner assignment/checkout failing due to user lookup mismatches
    -v1.0.0 assumed email matches Snipe-IT users. v2.0.0 implements multi-strategy user lookup (UPN/username/displayName) to handle LDAP-only username setups and missing email fields.

4. Crashes from unexpected Snipe-IT responses
    -v1.0.0 assumed specific JSON shapes. v2.0.0 is defensive when parsing API replies and logs helpful diagnostics.


## Breaking changes / Important migration notes

1. asset_tag now set to serial on create 
    —if your org previously used different asset tag scheme (e.g., numeric sequences generated in Snipe-IT), this change may cause duplicated asset_tags if you import devices that already exist. Recommendation: clean existing Snipe-IT records or run the script in --dry-run to observe effects first.

2. The script will not overwrite existing asset_tag and serial 
    —if you rely on the script to set new tags later, that behavior will not happen. You must change tags manually in Snipe-IT or add code to overwrite (not recommended).

3. User matching behavior changed 
    —ensure Snipe-IT users are discoverable by at least one of the matching strategies (UPN/username/display name). If your Snipe-IT records lack email/UPN, the script will attempt username/displayName fallback.

4. Status label names 
    —script searches labels by name. If your Snipe-IT uses different label names, add mapping or adjust labels to Pending, Ready to Deploy, Deployed (or modify the script constants).


## Developer notes (for maintainers)

1. Functions added/changed:

    -find_snipe_user() — robust user lookup with logging.

    -get_status_id_by_name() — searches statuslabels and prefers exact matches.

    -create_device_in_snipe_it() — sets asset_tag → serial by default and performs initial checkout if assigned_user present.

    -update_device_in_snipe_it() — performs narrow updates (name, assigned_user, optionally status) and avoids modifying asset_tag or serial.

2. Logging: verbose mode prints candidate user matching, status lookups, API responses (careful: do not enable verbose in CI with secrets visible).