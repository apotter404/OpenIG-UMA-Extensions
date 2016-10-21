# OpenIG-UMA-Extensions

OpenIG UMA Service and Filter Extensions for: <br />
1. Realm support <br />
2. Extend OpenIG-UMA REST endpoint <br /> 
3. User friendly UMA Resource name <br />
4. Persisting UMA RS id and PAT in OpenDJ <br />
5. Authentication for OpenIG-UMA REST endpoints using PAT <br />
6. //TODO Automatic refresh of PAT  <br />


Pre-requisites :
================
1. OpenAM has been installed and configured. Sample routes in this example use OpenAM realm '/employees'. 
2. OpenAM UMA service has been configured as specified here: https://backstage.forgerock.com/#!/docs/openam/13.5/admin-guide#configure-uma
3. OpenIG has been installed and configured as UMA RS as specified here: https://backstage.forgerock.com/#!/docs/openig/4.5/gateway-guide#chap-uma 
4. Maven has been installed and configured.


OpenDJ UMA RS store Installation & Configuration:
=================================================
1. Install OpenDJ under /opt/opendjis. Refer https://backstage.forgerock.com/#!/docs/opendj/3.5/install-guide#command-line-install <br />
   Setup params: <br />
   ============= <br />
   * Root User DN:                  cn=Directory Manager
   * Password                       cangetindj
   * Hostname:                      opendj.example.com
   * LDAP Listener Port:            3389
   * Administration Connector Port: 4444
   * SSL/TLS:                       disabled
   * Directory Data:                Backend Type: JE Backend, 
                                    Create New Base DN dc=openig,dc=forgerock,dc=org
   * Base DN Data: Only Create Base Entry (dc=openig,dc=forgerock,dc=org)


OpenIG Configuration:
=====================
1. Build OpenIG-UMA extension by running 'mvn clean install'. This will build openig-uma-ext-1.0.jar under /target directory.
2. Stop OpenIG. 
3. Copy openig-uma-ext-1.0.jar to <OpenIG-TomcatHome>/webapps/ROOT/WEB-INF/lib
4. Copy openig/config/routes/01-uma.json to OpenIG routes directory, Some details on this route: <br />
   * UmaServiceExt config, we can configure LDAP detials here:
   ```
       {
         "name": "UmaServiceExt",
         "type": "UmaServiceExt",
         "config": {
           "protectionApiHandler": "ClientHandler",
           "authorizationServerUri": "http://openam135.sample.com:8080/openam/",
   	    "realm": "/employees",
           "clientId": "OpenIG_RS",
           "clientSecret": "password",
           "ldapHost": "192.168.56.122",
           "ldapPort": 3389,
           "ldapAdminId": "cn=Directory Manager",
           "ldapAdminPassword": "cangetindj",
           "ldapBaseDN": "dc=openig,dc=forgerock,dc=org"
         }
       }
   ```
   * UmaFilterExt config, we can configure scope required for this filter here:
   ```
        {
          "type": "UmaFilterExt",
          "config": {
            "protectionApiHandler": "ClientHandler",
            "umaService": "UmaServiceExt",
            "scopes" : [
              "http://login.example.com/scopes/view"
            ]
          }
        }
   ```
      
OpenIG Use Cases testing:
=========================
1. /uma folder updates are required in openig-doc-4.5.0-jar-with-dependencies.jar. Unpack this jar and replace /uma folder contents with /openig-doc-ext/uma files. If required; update common.js configs like OpenAM url etc. 
2. Execute this for testing OpenIG-UMA usecases: https://backstage.forgerock.com/#!/docs/openig/4.5/gateway-guide#uma-trying-it-out

OpenIG-UMA REST endpoints:
==========================
All below REST endpoints require valid PAT in Authorization header. 
 
1. Create share. UMA shares need to have unique uri and name. Note that this restriction is per uid per realm per OAuth Client. In other words if user with uid 'alice' (in realm /employees and and using OAuth Client: OpenIG_RS) has created UMA share with name: app1 and uri: /app1, then she can't create share with name: app1 and uri /app2 (or name: app2 and uri /app1) but alice in /customer realm can create such share.
```
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer <Valid PAT>" -d '{
	 "uri" : "/photoAlbum",
     "name" : "PhotoAlbum",
     "scopes" : [
         "http://photoz.sample.com/scopes/view",
         "http://photoz.sample.com/scopes/all"
     ],
     "type" : "http://photoz.sample.com"
 }' "http://<OpenIG-Host:Port>/openig/api/system/objects/router-handler/routes/01-uma/objects/umaserviceext/share?_action=create"
 
{
  "_id": "b3400172-90a5-4c7f-8c0f-c5b35ea7e11c",
  "resourceURI": "/photoAlbum",
  "user_access_policy_uri": "http://openam135.sample.com:8080/openam/XUI/?realm=/employees#uma/share/1c62792d-4a88-455b-bf7d-9c6ac1433bc55",
  "pat": "8751ca72-3925-4528-b14c-5750f12ab0ac",
  "resource_set_id": "1c62792d-4a88-455b-bf7d-9c6ac1433bc55",
  "userId": "alice",
  "realm": "/employees",
  "client_id": "OpenIG_RS"
} 
```

2. Read all shares:
```
curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer <Valid PAT>" "http://<OpenIG-Host:Port>/openig/api/system/objects/router-handler/routes/01-uma/objects/umaserviceext/share?_queryFilter=true"
 
{
  "result": [
    {
      "_id": "4b179c9f-823a-4e62-bc7a-fe51929d41a7",
      "resourceURI": "/login",
      "user_access_policy_uri": "http://openam135.sample.com:8080/openam/XUI/?realm=/employees#uma/share/cae0104a-668a-4f07-948e-0a843f8c89823",
      "pat": "38909fe9-c18e-45c1-9b46-509313e99144",
      "resource_set_id": "cae0104a-668a-4f07-948e-0a843f8c89823",
      "userId": "alice",
      "realm": "/employees",
      "client_id": "OpenIG_RS"
    }
  ],
  "resultCount": 1,
  "pagedResultsCookie": null,
  "totalPagedResultsPolicy": "NONE",
  "totalPagedResults": -1,
  "remainingPagedResults": -1
}
```

3. Read specific share. Note that this requires <OpenIG-ResourceId> in REST URL. 
```
curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer <Valid PAT>" "http://<OpenIG-Host:Port>/openig/api/system/objects/router-handler/routes/01-uma/objects/umaserviceext/share/<OpenIG-ResourceId>"

{
  "_id": "4b179c9f-823a-4e62-bc7a-fe51929d41a7",
  "resourceURI": "/login",
  "user_access_policy_uri": "http://openam135.sample.com:8080/openam/XUI/?realm=/employees#uma/share/cae0104a-668a-4f07-948e-0a843f8c89823",
  "pat": "38909fe9-c18e-45c1-9b46-509313e99144",
  "resource_set_id": "cae0104a-668a-4f07-948e-0a843f8c89823",
  "userId": "alice",
  "realm": "/employees",
  "client_id": "OpenIG_RS"
}
```

4. Delete specific share. Note that this requires <OpenIG-ResourceId> in REST URL.
```
curl -X DELETE -H "Content-Type: application/json" -H "Authorization: Bearer <Valid PAT>" -d '' "http://<OpenIG-Host:Port>/openig/api/system/objects/router-handler/routes/01-uma/objects/umaserviceext/share/<OpenIG-ResourceId>"

{
  "_id": "4b179c9f-823a-4e62-bc7a-fe51929d41a7",
  "resourceURI": "/login",
  "user_access_policy_uri": "http://openam135.sample.com:8080/openam/XUI/?realm=/employees#uma/share/cae0104a-668a-4f07-948e-0a843f8c89823",
  "pat": "38909fe9-c18e-45c1-9b46-509313e99144",
  "resource_set_id": "cae0104a-668a-4f07-948e-0a843f8c89823",
  "userId": "alice",
  "realm": "/employees",
  "client_id": "OpenIG_RS"
}
```


* * *

Copyright © 2016 ForgeRock, AS.

This is unsupported code made available by ForgeRock for community development subject to the license detailed below. The code is provided on an "as is" basis, without warranty of any kind, to the fullest extent permitted by law. 

ForgeRock does not warrant or guarantee the individual success developers may have in implementing the code on their development platforms or in production configurations.

ForgeRock does not warrant, guarantee or make any representations regarding the use, results of use, accuracy, timeliness or completeness of any data or information relating to the alpha release of unsupported code. ForgeRock disclaims all warranties, expressed or implied, and in particular, disclaims all warranties of merchantability, and warranties related to the code, or any service or software related thereto.

ForgeRock shall not be liable for any direct, indirect or consequential damages or costs of any type arising out of any action taken by you or others related to the code.

The contents of this file are subject to the terms of the Common Development and Distribution License (the License). You may not use this file except in compliance with the License.

You can obtain a copy of the License at https://forgerock.org/cddlv1-0/. See the License for the specific language governing permission and limitations under the License.

Portions Copyrighted 2016 Charan Mann
