# OctoRPKI Fails to Refresh Snapshot and Delta When Serial is Poisoned, Causing Persistent Outdated VRPs
**Affected target, feature, or URL:**
The affected target is the octorpki component in https://github.com/cloudflare/cfrpki.

**Description of problem:**
OctoRPKI fails to properly refresh snapshot and delta files from repositories when the `serial` value in `notification.xml` is abnormally large.

This causes the RP to incorrectly assume that no updates are needed, even if the actual contents of the repository (e.g., new ROAs) have changed. As a result, OctoRPKI remains stuck in a stale state and continues to serve outdated VRPs indefinitely.

The root cause lies in incomplete validation logic within OctoRPKI's RRDP sync flow:
- There is no check to determine whether the snapshot hash has changed.
- A poisoned `serial` value (e.g., 999999999999999 or int64 max) will prevent future snapshot fallback or delta fetches, even after the repository is fixed.
- OctoRPKI does not emit a warning or error, silently skipping reprocessing.

This behavior can lead to persistent misvalidation of BGP routes under ROV, ultimately threatening routing correctness and availability.

OctoRPKI remains locked in a permanently stale state unless its cache is manually cleared or the session ID is forcibly changed.

**Steps to reproduce:**
1. In the `notification.xml` file, change the `serial` attribute to the maximum 64-bit signed integer value: 9223372036854775807.

2. Execute octorpki with the following command: `sudo ./octorpki -allow.root -mode oneoff -tal.root /home/zclin/TAL/ta.tal -output.roa /var/lib/krill/test/vrps/vrps.json -loglevel debug`. After execution, inspect octorpki’s cache file `rrdp.json`. We will find that the serial value has been updated to: 9223372036854775807.

3. Use Krill's web interface to add a new ROA object to the repository.
This operation will trigger a regeneration of the `notification.xml` file.
By inspecting the updated `notification.xml`, confirm that the file has been modified accordingly, and the `serial` value has been restored to a normal value.

4. Re-run the following command: `sudo ./octorpki -allow.root -mode oneoff -tal.root /home/zclin/TAL/ta.tal -output.roa /var/lib/krill/test/vrps/vrps.json -loglevel debug`. Then, inspect the generated `vrps.json` file. Notice that the number of validated ROAs remains unchanged — the newly added ROA has not been included in the validation results.

5. The root cause of this issue lies in the logic within `rrdp.go`.
At line 304, the following loop is used: `for serial = lastSerial + 1; serial <= curSerial && curSerial - lastSerial > 0; serial++ {`, If the cached `lastSerial` value is greater than `curSerial`, the loop body will not be executed. As a result, even if new ROAs are added and `delta.xml` has been updated, octorpki will not fetch the delta file.
In addition, at line 240, the following condition is used: `lastSerial == 0 || lastSessionID != curSessionID || missingFiles`. Since none of the conditions are met — `lastSerial` is not zero, the `session_id` has not changed, and `delta.xml` is not missing — octorpki will also skip fetching the snapshot.
Together, these conditions cause octorpki to stuck in a stale state, even when the repository is updated with new ROAs.

**Reproduction video:**  
https://github.com/user-attachments/assets/36218145-ce08-4526-9d61-4deea3bca031



## Summary:
OctoRPKI becomes stuck in a stale state when the RRDP `notification.xml` contains an abnormally large `serial` value. Even if the repository is subsequently updated and the serial is restored to a normal value, OctoRPKI will **refuse to re-fetch either the snapshot or any deltas**, resulting in the continuous use of **permanently stale and outdated VRPs**. **This behavior may impact the integrity of VRP data and undermine the availability of Route Origin Validation (ROV) as a defense against route hijacking.**
