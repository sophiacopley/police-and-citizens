# New threats added on branch data-flows

Threats added to `ThreatDragonModels/login register flow/login register flow.json` since this branch diverged from `main`.

| # | Title | Type | Added to | Description | Mitigation |
|---|-------|------|-----------|-------------|------------|
| 186 | Citizen register request is changed in transit | Tampering | get_register_request | Request URL query parameters or body is tampered with by a bad actor e.g., injecting a malicious redirect or return parameter. | - Use HTTPS (or another encrypted protocol) so data is encrypted in transit |
| 187 | Citizen login request is changed in transit | Tampering | get_login_request | See #186 |  |
| 188 | Flooding the network with traffic | Denial of service | get_login_request | See #15 |  |
| 189 | Login process response is changed in transit | Tampering | get_login_response | See #16 |  |
| 190 | Flooding the network with traffic | Denial of service | get_login_response | See #15 |  |
| 191 | Flooding the network with traffic | Denial of service | submit_login_request | See #15 | Provide remediation for this threat or a reason if status is N/A |
| 192 | Flooding the network with traffic | Denial of service | submit_login_response/session_cookie | See #15 | Provide remediation for this threat or a reason if status is N/A |
| 193 | Flooding the network with traffic | Denial of service | redirect_to_login_response | See #15 |  |
| 194 | Flooding the network with traffic | Denial of service | verify_email_request | See #15 |  |
| 195 | Flooding the network with traffic | Denial of service | get_register_response | See #15 |  |
| 196 | Flooding the network with traffic | Denial of service | submit_register_response | See #15 |  |
| 197 | Citizen information disclosed to a bad actor | Information disclosure | submit_login_request | See #18 |  |
| 199 | Citizen login data tampered with in transit | Tampering | submit_login_request | A Bad Actor could tamper with the login credentials that the citizen supplied and log them into an account that is not theirs. | - Use HTTPS (or another encrypted protocol) to send data over the network |
| 200 | Citizen session cookie exposed in transit | Information disclosure | submit_login_response/session_cookie | A Bad Actor could observe the session cookie assigned to the Citizen on log-in and use the session-cookie to perform actions on behalf of the Citizen. | - Use HTTPS (or another encrypted protocol) to send data over the network - Rotate session Cookies often  |
| 201 | Login process response altered in transit | Tampering | submit_login_response/session_cookie | If a Bad Actor observed and altered the response from the login process to a different session cookie. The Citizen may be logged into an account owned by the Bad Actor. The Bad Actor could also alter the response to redirect the Citizen to a malicious website or to a spoofed version of SecureReports. | - Use HTTPS (or another encrypted protocol) to send data over the network |
| 202 | Citizen verify request is changed in transit | Tampering | verify_email_request | A Bad Actor could tamper with the verification code in the request to be invalid and prevent the Citizen from verifying their account. | - Use HTTPS (or another encrypted protocol) to send data over the network |
| 203 | Verify Email process response is changed during transit | Tampering | redirect_to_login_response | See #16  |  |
