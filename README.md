# Error Hadling - Mulesoft
![](https://docs.mulesoft.com/mule-runtime/4.4/_images/error-handling-try.png) *Example of the On Error Propagate connector*
## Case of Study
- Use case: Clients can fetch actors details from a movie database on actor ID.
    http://localhost:8081/fetch-actor?actorid=100
    - Must validate ``actorid`` is a number or not.
    - Example response:
        ```
        Response: 200, OK
        ```
        ```json
        {
            "actorid": 100,
            "actorFistName": "test",
            "actorLastName": "something",
            "last_update": "2006-02-15T04:34:33"
        }
        ```
        ---
        - Correctly handle error when querying to http://localhost:8081/fetch-actor?actorid=100er or invalid number:
            ```
            400, Bad Request
            ```
            ```json
            {
                "status": 400,
                "message": "actorid should be number"
            }
            ```
        ---
    - Must validate there are no `DB:CONNECTIVITY` errors due to Database service is down.
        ```
        404001 Actor not Found
        ```
        ```json
        {
            "status": 404001,
            "message": "Actor Not Found"
        }
        ```
## Configuration the Mule application flow before implementing *On Error Propagate* connector
- Steps:
    - Prepare a new Mule Project with the following Mule event trigger and processors:
        - Note: *all global configurations will be set under the Secure Properties Global Configuration Elements within* **Secure Properties Config** to encrypt database password and set http path and port as well as database user etc.
            - Under `./src/main/resources` create 2 files:
                1. `temp.properties`, will set the **encryption algorithm** for the password property & the keyword *(16 alphanumeric digit)* to **decrypt** the password contained within `dev-prop-secure.yaml`; this is configured & **open with** with *Mule Properties Editor*:

                    ![](https://dz2cdn1.dzone.com/storage/temp/13775202-8.png)
                2. `dev-prop-secure.yaml`, will contain details (properties) for **http** & **database** connection.
                ```yaml
                http:
                    port: "8081"
                    host: "localhost"
                    path: "fetch-actor"
                db: 
                    user: "test"
                    host: "localhost"
                    port: "3306"
                    # password value is encrypted now 👀
                    password: "![4Sry/LV7HBOAyNNHOLrjWg==]"
                ```
            - At the Anypoint Studio Canvas, on the main `xml` file of the flows for our Mule app, **Add from the Anypoint Studio Palette ⚙ -> Modules ->  ➕ Add new module from Exchange -> Mule Secure Configuration Property Extension**

                ![](https://media-exp1.licdn.com/dms/image/C5112AQGpLFE0LxeCsQ/article-inline_image-shrink_1000_1488/0/1571989992417?e=1649894400&v=beta&t=tfeGnJ_R60BsZB1A1afInkBSczHOicQJehmCkt-rCCA)
                - *Apply -> Apply and Close* 
            - At main Anypoint Studio menu -> *Run -> Run Configurations... -> Arguments:*
                ![](https://miro.medium.com/max/1400/1*kWe0zI8SPkGFe9ZCcPxMPg.png)
                |Arguments|definition/explanation|
                |-|-|
                |``-Denv=dev-secure``| `-D` is the declaration of a new *Program variable*, `env` is the name of the variable, `dev-secure` is the value set to later reference the configuration properties file (`dev-prop-secure.yaml`).|
                |``-Denc.key=sixteenDigitPassword``| `-D` is the declaration of a new *Program variable*, `enc.key` is the name of the variable for the password encrypted key-value pair, `sixteenDigitPassword` is the value set to **decrypt** the referenced password within db.password at the`dev-prop-secure.yaml`.|

                - *Apply -> Run -> Save all*
                ---

    1. Add an **HTTP Listener** mule event trigger with the following *connector configuration*:
        ```xml
        <http:listener doc:name="Listener" doc:id="236986d1-b734-4b4c-bfe9-a8272b5590c2" config-ref="HTTP_Listener_config" path="${secure::http.path}"/>
        ```

        |Port|Explanation|
        |-|-|
        |`${secure::http.port}`|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `http.port` ("8081") references to the `dev-prop-secure.yaml` environment properties configuration file.|
        ---
        General 
        |Path|Explanation|
        |-|-|
        |``${secure::http.path}``|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `http.path` ("fetch-actor") references to the `dev-prop-secure.yaml` environment properties configuration file.|
    2. Add a *Is number* - Validation mule event processor, to ensure query Parameter value is an *Integer* number; with the following configuration:
        ```xml
        <validation:is-number doc:name="Is number" doc:id="8b2ad1c2-0f4a-4ba3-87b1-8f4657d4a69f" value="#[attributes.queryParams.actorid]" numberType="INTEGER" message="actorid must be a Number"/>
        ``` 
        |attributes from `<validation:is-number>` XML tag| Explanation |
        |-|-|
        |``value="#[attributes.queryParams.actorid]"``| Uses DataWeave transform language to assign a value from the mule event message attributes and take it from the query Paramater: `?actorid=100` to access an number identificator, can also access a not desired type of digit such as a name.|
        |``numberType="INTEGER"``|Sets the data type to be validated.|
        |``message="actorid must be a Number"``|Sets an Error message if validation results to false.|

    3. Add a **Select** - Database mule event processor with the following *connector configuration*:
        
        *Global elements configuration:*
        ```xml
        <db:config name="Database_Config" doc:name="Database Config" doc:id="25e5f98f-c434-47d4-8bd4-fe5e10cc6bcf" >
            <db:my-sql-connection host="${secure::db.host}" port="${secure::db.port}" user="${secure::db.user}" database="sakila" password="${secure::db.password}"/>
        </db:config>
        ```

        |Host|Explanation|
        |-|-|
        |`${secure::db.host}`|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `db.host` ("localhost") references to the `dev-prop-secure.yaml` environment properties configuration file.|
        
        |Port|Explanation|
        |-|-|
        |`${secure::db.port}`|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `db.port` ("3306") references to the `dev-prop-secure.yaml` environment properties configuration file.|
        
        |User|Explanation|
        |-|-|
        |`${secure::db.host}`|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `db.user` ("test") references to the `dev-prop-secure.yaml` environment properties configuration file.|
        
        |User|Explanation|
        |-|-|
        |`${secure::db.password}`|`${}` are used to reference a value, in this case `secure::` references that a Mule Secure Properties extension and configuration is been used, while `db.password` ("![4Sry/LV7HBOAyNNHOLrjWg==]" - encrypted value) references to the `dev-prop-secure.yaml` environment properties configuration file and the `temp.properties` file.|

        *Inside the flow:*
        - Retrieves from `sakila.actor` SQL table and database, one actor that is identified by the `actorid` provided within the URL: http://localhost:8081/fetch-actor?actorid=100, which is then processed within the Mule event as an attribute query parameters: *attribute.queryParams*.
        ```xml
        <db:select doc:id="12ceaae7-5790-4c9c-a4de-9746e42097f1" config-ref="Database_Config">
            <db:sql ><![CDATA[SELECT * FROM actor
            WHERE actor_id = :actorID]]></db:sql>
            <db:input-parameters ><![CDATA[#[{
                actorID: attributes.queryParams.actorid
            }]]]></db:input-parameters>
        </db:select>
        ```
        |`config-ref`|Explanation|
        |-|-|
        |``config-ref="Database_Config"``|References to the *Global Elements Configuration*|

    4. Add a **Transform Message** mule event processor, which parses the result from querying database from a Java response to a more client-friendly format that could be a JSON, or XML response; which will contain the following configurations.
        ```xml
        <ee:transform doc:name="Transform Message" doc:id="0603f5e3-9234-4b8c-a211-d6fd233c0ccb" >
			<ee:message >
				<ee:set-payload ><![CDATA[%dw 2.0
                output application/json
                ---
                payload[0]]]></ee:set-payload>
			</ee:message>
		</ee:transform>
        ```
    -----
    **Note:** *When querying from a REST client an Identificator for actors actorid as a number data type and within the database identificators records with this URL:http://localhost:8081/fetch-actor?actorid=100, the result should be:*
    
    ``200 - HTTP Status code``
    ```json
    {
        "last_update": "2006-02-15T04:34:33",
        "last_name": "GUINESS",
        "actor_id": 1,
        "first_name": "PENELOPE"
    }
    ```
    - When querying from the REST client with http://localhost:8081/fetch-actor?actorid=100ere adding more alphabetic digits produces this result:

        ``500 Server Error - HTTP Status code``
        ```json
        "actorid must be a Number"
        ```
        **Response is not correctly handled, it requires to be this way:**
            
        ```
        400, Bad Request
        ```
        ```json
        {
            "status": 400,
            "message": "actorid should be number"
        }
        ```
    - Another error not correctly handled by the Mule application could be when MySQL server is down or the connection has been lost, the response after querying from the REST client to the URL: http://localhost:8081/fetch-actor?actorid=100 will produce the following error:

        ```
        500 Server Error - HTTP Status Code
        ```

        ```json
        Cannot get connection for URL jdbc:mysql://localhost:3306/sakila?logger=org.mule.extension.db.api.logger.MuleMySqlLogger$$EnhancerByCGLIB$$42140f7b : Communications link failure

        The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
        ```

## On Error Components
*src: https://docs.mulesoft.com/mule-runtime/4.4/on-error-scope-concept*
![](https://docs.mulesoft.com/mule-runtime/4.4/_images/mruntime-on-error-propagate.png) *On Error Components*

- When an error occurs in a Mule app, an Error Handler component routes the error to the first On-Error component (On Error Continue or Error Propagate) configuration that matches the Mule error, such as ``HTTP:NOT_FOUND`, `DB:CONNECTIVITY`, or `ANY` (the default for all Mule errors). If no error handling is configured for the error, the app follows a default error handling process.

![](https://docs.mulesoft.com/mule-runtime/4.4/_images/mruntime-error-handler.png)*Error processors in Anypoint Studio's Mule Palette*

## Types of Error Handling in Mule 4
- Helps sending user-friendly messages.
- An error bad handled terminates the execution of Mule application whenever an exception rises.
- An error correctly handled allow us to continue the flow or alternative execution of code that handles the situation.


1. On error propagate.
2. On error continue.
3. Raise error.
4. Error handler.
- Try

    ## 1. On Error Propagate
    - Executes but propagates the error to a higher level, such as a containing scope (for example, to a Flow that contains a Try scope where the error occurs).

    - *On Error Propagate* connector must be used on a delimited scope such as: Try or flow error handling.
        
        ![](https://blogs.mulesoft.com/wp-content/uploads/Error-Propagate-scope.png)

        - At *Error Handling* On Error Propagate will identify all posible erro types that may come from the flow.

## On Error Propagate, error caused by *URL having alphanumeric digits*

- Create 2 variables: **statusCode** & **reasonPhrase**; which contain the HTTP Status code for the **Handle Response** we want to handle, as well as the **reason for error** we want to set.

```xml
<error-handler >
    <on-error-propagate enableNotifications="true" logException="true" doc:name="On Error Propagate" doc:id="4ca27ab2-0623-4ab5-9b8f-500bbb319cc8" type="VALIDATION:INVALID_NUMBER">
        <ee:transform doc:name="Transform Message" doc:id="4a5c5a1c-5238-4b65-977a-2b2597a048e2" >
            <ee:message >
                <ee:set-payload ><![CDATA[%dw 2.0
                    output application/json
                    ---
                    {
                        "status": 400,
                        "message": error.description
                    }]]></ee:set-payload>
            </ee:message>
            <ee:variables >
                <ee:set-variable variableName="statusCode" ><![CDATA[400]]></ee:set-variable>
                <ee:set-variable variableName="reasonPhrase" ><![CDATA["Bad Request"]]></ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```
- `error.description` will log out the description of the Message for *Error options* defined at *Is number - Validation* connector.

|Select the error types|
|-|
|VALIDATION:INVALID_NUMBER|

- Can handle the response using a transform message processor that gives back to the user a friendlier message of the problem and a tip on how it could be solve it.

- Modify *Error Responses*  at **HTTP Listener** source connector to log to REST client the requiered HTTP Status code & Reason Phrase:

    ![](https://docs.mulesoft.com/http-connector/1.6/_images/http-listener-9.png)*Sample menu configuration of HTTP Listener on the Responses*
    - Steps:
        - Under *HTTP Listener tab -> Error Response* set the following configurations:
            |Error Response|Value|
            |-|-|
            |Body: | `#[payload]`|
            |Status code: | `#[vars.statusCode]`|
            |Reason phrase : | `#[vars.reasonPhrase]`|

        *XML tag for HTTP Listener connector with updated variables should look like the follwoing:*

        ```xml
        <http:listener doc:name="Listener" doc:id="bdca3b82-355e-4a97-9e13-18c75eebf431" config-ref="HTTP_Listener_config" path="${secure::http.path}">
            <http:error-response statusCode="#[vars.statusCode]" reasonPhrase="#[vars.reasonPhrase]">
                <http:body ><![CDATA[#[payload]]]></http:body>
            </http:error-response>
        </http:listener>
        ```


### Result of the Handled Error:

- When URL is queryed to: http://localhost:8081/fetch-actor?actorid=100er, the **handle error's Response is set to**:
    ```json
    {
    "status": 400,
    "message": "actorid must be a Number"
    }
    ```

## On Error Propagate, Database service is down.
- Add to the flow an *On Error Propagate* connector
    - Add a *Transform Message* connector to the flow and configure response error message for the client:
        ```
        %dw 2.0
        output application/json
        ---
        ```
        ```json
        {
            "status": 500,
            "message": "Internal server error"
        }
        ```
----



## 2. On Error continue
- Allows to continue the execution of the flow components even though it has found an error.
- **Try** connector allow us to handle the error per Mule processor:
    ![](https://docs.mulesoft.com/mule-runtime/4.3/_images/error-handling-try-scope.png)*Try Scope*
    - Enables to handle errors that may occur when attempting to execute any of the components inside the Try scope.

- Steps to use *Try* & *On Error Continue*:
    ```xml
    <try doc:name="Try" doc:id="2a063879-775b-42c9-9736-3b74cd14ea26" >
        <db:select doc:id="12ceaae7-5790-4c9c-a4de-9746e42097f1" config-ref="Database_Config">
			<db:sql><![CDATA[SELECT * FROM actor
                WHERE actor_id = :actorID]]></db:sql>
			<db:input-parameters><![CDATA[#[{
	            actorID: attributes.queryParams.actorid
            }]]]></db:input-parameters>
		</db:select>
        <error-handler >
            <on-error-continue enableNotifications="true" logException="true" doc:name="On Error Continue" doc:id="fd569a5e-5156-40c0-b232-b27bcb03d10f" type="ANY">
                <logger level="INFO" doc:name="Logger" doc:id="319ac657-fc2d-4af8-a40d-62c013ce17a5" message="Database error ocurred"/>
            </on-error-continue>
        </error-handler>
    </try>
    ```
    - Add to the probable failure component *Select* a *Try* processor within the flow.
    - The probable failure component must be inside the *Try component*.
    - Add to the Try - Error Handling scope a *On error Continue* component
        - Set the selection of the error type to *Any*
        - Add within *On error continue* a *Logger* processor
*Note: When querying to http://localhost:8081/fetch-actor?actorid=100 when database service is down, now the error is handle by the Try scope and not the global error neither by Error Propagate, which allow us to continue the flow execution that produces the follwoing output response to the REST client:*
    ```
    "�"
    ```
- Logs this to the Mule console:
    ```
    org.mule.runtime.core.internal.processor.LoggerMessageProcessor: Database error ocurred
    ```
---

## Error Handler


## Differences between Handling Errors
|`On Error Propagate`|`On Error Continue`|``Raise Error``|`Error Handler`|`Try`|
|-|-|-|-|-|
|Whenever it founds an error, it stops the execution, then it responds back the error response to the source |Will continue the remaining flow execution|``Raise Error``|`Error Handler`|Allows to handle errors per component or Mule event processors, it also has an Error Handling of that subFlow, which handles the error before the Global Error from the main Flow.|