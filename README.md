# ServiceNowLearn – FTP User Integrations

This project demonstrates how to integrate **ServiceNow** with external systems using **REST APIs**, including **Atlassian JIRA** and an **LDAP Test Server**.  
Additionally, it showcases **E-Bonding** with another ServiceNow instance.

---

## Features
- Integrated **ServiceNow ↔ JIRA** with REST APIs for seamless issue synchronization.
- Achieved inbound integration with an **online LDAP Test Server**.
- Completed **E-Bonding** with another ServiceNow instance.

---

## LDAP Integration Steps

1. Navigate to **System LDAP → Create New Server**.
2. Accept the default `Active Directory` as the LDAP server type.
3. Enter a meaningful **Server Name** (e.g., `LDAP Server`).
4. Copy **Server + Port** from the LDAP Test Server into the **Server URL** field.
5. Enter **Starting Search Directory** (e.g., `dc=example,dc=com`).
6. Click **Submit**.
7. On the LDAP Server page, set:
   - **Login distinguished name** → `cn=read-only-admin,dc=example,dc=com`
   - **Login password** → use test server password.
8. Note values for **Connection Timeout**, **Read Timeout**, and **Listen Interval**.
9. Scroll to **Users**:
   - Remove filter and RDN.
   - Add a custom filter if needed (e.g., `(uid=e*)` for users starting with "e").
10. Click **Test Connection** (should succeed).
11. Click **Browse** (should list users).
12. Select **Data Source → Load All Records**.
13. Create a **Transform Map** (e.g., `TM LDAP to SYS_USER Table`).
14. Map source fields to `sys_user` table fields.
15. Run **Transform** → `sys_user` table should be populated with LDAP test users.

---

## JIRA Integration Steps

1. **Create Basic Auth Profile** (`sys_auth_profile_basic`):
   - Name: `JIRA PersonalAccessToken BasicAuthProfil`
   - Username: Atlassian Jira account email
   - Password: Personal Access Token (PAT)
   - 🔧 Workaround: Increase password column length from 255 → 500.
2. Go to **System Web Services → Outbound → REST Message** → **New**.
   - Name: `JIRA Outbound Integration`
   - Description: `Integrating with Atlassian JIRA`
   - Endpoint URL: `https://<yourservername>.atlassian.net/`
   - Auth Type: **Basic** → Select the above Basic Auth profile.
3. Under **HTTP Request → HTTP Headers**:
   - Name: `Content-Type`
   - Value: `application/json`
4. Test **Default GET** → expect HTTP `200`.
5. Create **POST Method**:
   - Name: `Create Incident on JIRA`
   - Method: `POST`
   - Endpoint: `https://<yourservername>.atlassian.net/rest/api/2/issue/`
   - Authentication: Inherit from parent
   - Request Body:
     ```json
     {
       "fields": {
         "project": { "key": "JS" },
         "summary": "${short_description}",
         "description": "${description}",
         "issuetype": { "name": "Task" }
       }
     }
     ```
6. Auto-generate variables → Preview Script Usage.
7. Create a **Business Rule** (After Insert) to invoke the REST Message.

---

## Business Rule Script

```javascript
(function executeRule(current, previous /*null when async*/ ) {
    try {
        // Build description safely
        var descParts = [];
        if (current.description) descParts.push(current.description.toString());
        if (current.caller_id) descParts.push(current.caller_id.toString());
        if (current.number) descParts.push(current.number.toString());
        var description = descParts.join(' | ');

        // REST Message
        var request = new sn_ws.RESTMessageV2('JIRA Outbound Integration', 'Create Incident on JIRA');
        request.setStringParameterNoEscape('description', description);
        request.setStringParameterNoEscape('short_description', current.short_description ? current.short_description.toString() : '');

        // Execute
        var response = request.execute();
        var httpStatus = response.getStatusCode();
        var responseBody = response.getBody();

        gs.info("JIRA Integration → Status: " + httpStatus);
        gs.info("JIRA Integration → Body: " + responseBody);

        // Parse JIRA response JSON
        var responseJSON = JSON.parse(responseBody);

        // Update correlation_id with Jira issue ID
        if (responseJSON && responseJSON.id) {
            current.setValue('correlation_id', responseJSON.id);
            current.update();
        }

    } catch (ex) {
        gs.error("JIRA Integration Error: " + ex.message);
    }
})(current, previous);
```
---
## Outcome

- LDAP Integration successfully loaded test users into sys_user.

- JIRA Integration allowed incidents created in ServiceNow to automatically sync with JIRA.

- Demonstrated secure authentication with Basic Auth + PAT.

- Verified two-way connectivity with successful API calls.
---

## Author

- Anasuya Rampalli
