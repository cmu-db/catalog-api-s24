# Catalog API specification
## Endpoints
  ### /v1/config:
     get:
      tags:
        - Configuration API
      summary: List all catalog configuration settings
      operationId: getConfig
      parameters:
        - name: warehouse
          in: query
          required: false
          schema:
            type: string
          description: Warehouse location or identifier to request from the service
      description:
        "
        All REST clients should first call this route to get catalog configuration
        properties from the server to configure the catalog and its HTTP client.
        Configuration from the server consists of two sets of key/value pairs.

        - defaults -  properties that should be used as default configuration; applied before client configuration

        - overrides - properties that should be used to override client configuration; applied after defaults and client configuration


        Catalog configuration is constructed by setting the defaults, then client-
        provided configuration, and finally overrides. The final property set is then
        used to configure the catalog.


        For example, a default configuration property might set the size of the
        client pool, which can be replaced with a client-specific setting. An
        override might be used to set the warehouse location, which is stored
        on the server rather than in client configuration.


        Common catalog configuration settings are documented at
        https://iceberg.apache.org/configuration/#catalog-properties
        "
      responses:
        200:
          description: Server specified configuration values.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CatalogConfig'
              example: {
                "overrides": {
                  "warehouse": "s3://bucket/warehouse/"
                },
                "defaults": {
                  "clients": "4"
                }
              }
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  
      tags:
        - OAuth2 API
      summary: Get a token using an OAuth2 flow
      operationId: getToken
      description:
        Exchange credentials for a token using the OAuth2 client credentials flow or token exchange.


        This endpoint is used for three purposes -

        1. To exchange client credentials (client ID and secret) for an access token
           This uses the client credentials flow.

        2. To exchange a client token and an identity token for a more specific access token
           This uses the token exchange flow.

        3. To exchange an access token for one with the same claims and a refreshed expiration period
           This uses the token exchange flow.


        For example, a catalog client may be configured with client credentials from the OAuth2
        Authorization flow. This client would exchange its client ID and secret for an access token
        using the client credentials request with this endpoint (1). Subsequent requests would then
        use that access token.


        Some clients may also handle sessions that have additional user context. These clients would
        use the token exchange flow to exchange a user token (the "subject" token) from the session
        for a more specific access token for that user, using the catalog's access token as the
        "actor" token (2). The user ID token is the "subject" token and can be any token type
        allowed by the OAuth2 token exchange flow, including a unsecured JWT token with a sub claim.
        This request should use the catalog's bearer token in the "Authorization" header.


        Clients may also use the token exchange flow to refresh a token that is about to expire by
        sending a token exchange request (3). The request's "subject" token should be the expiring
        token. This request should use the subject token in the "Authorization" header.
      requestBody:
        content:
          application/x-www-form-urlencoded:
            schema:
              $ref: '#/components/schemas/OAuthTokenRequest'
      responses:
        200:
          $ref: '#/components/responses/OAuthTokenResponse'
        400:
          $ref: '#/components/responses/OAuthErrorResponse'
        401:
          $ref: '#/components/responses/OAuthErrorResponse'
        5XX:
          $ref: '#/components/responses/OAuthErrorResponse'

  ### /v1/{prefix}/namespaces:
    parameters:
      - $ref: '#/components/parameters/prefix'

    get:
      tags:
        - Catalog API
      summary: List namespaces, optionally providing a parent namespace to list underneath
      description:
        List all namespaces at a certain level, optionally starting from a given parent namespace.
        If table accounting.tax.paid.info exists, using 'SELECT NAMESPACE IN accounting' would
        translate into `GET /namespaces?parent=accounting` and must return a namespace, ["accounting", "tax"] only.
        Using 'SELECT NAMESPACE IN accounting.tax' would
        translate into `GET /namespaces?parent=accounting%1Ftax` and must return a namespace, ["accounting", "tax", "paid"].
        If `parent` is not provided, all top-level namespaces should be listed.
      operationId: listNamespaces
      parameters:
        - name: parent
          in: query
          description:
            An optional namespace, underneath which to list namespaces.
            If not provided or empty, all top-level namespaces should be listed.
            If parent is a multipart namespace, the parts must be separated by the unit separator (`0x1F`) byte.
          required: false
          allowEmptyValue: true
          schema:
            type: string
          example: "accounting%1Ftax"
      responses:
        200:
          $ref: '#/components/responses/ListNamespacesResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - Namespace provided in the `parent` query parameter is not found.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NoSuchNamespaceExample:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    post:
      tags:
        - Catalog API
      summary: Create a namespace
      description:
        Create a namespace, with an optional set of properties.
        The server might also add properties, such as `last_modified_time` etc.
      operationId: createNamespace
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateNamespaceRequest'
      responses:
        200:
          $ref: '#/components/responses/CreateNamespaceResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        406:
          $ref: '#/components/responses/UnsupportedOperationResponse'
        409:
          description: Conflict - The namespace already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceAlreadyExists:
                  $ref: '#/components/examples/NamespaceAlreadyExistsError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'

    get:
      tags:
        - Catalog API
      summary: Load the metadata properties for a namespace
      operationId: loadNamespaceMetadata
      description: Return all stored metadata properties for a given namespace
      responses:
        200:
          $ref: '#/components/responses/GetNamespaceResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - Namespace not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NoSuchNamespaceExample:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    head:
      tags:
        - Catalog API
      summary: Check if a namespace exists
      operationId: namespaceExists
      description:
        Check if a namespace exists. The response does not contain a body.
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - Namespace not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NoSuchNamespaceExample:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    delete:
      tags:
        - Catalog API
      summary: Drop a namespace from the catalog. Namespace must be empty.
      operationId: dropNamespace
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - Namespace to delete does not exist.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NoSuchNamespaceExample:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/properties:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'

    post:
      tags:
        - Catalog API
      summary: Set or remove properties on a namespace
      operationId: updateProperties
      description:
        Set and/or remove properties on a namespace.
        The request body specifies a list of properties to remove and a map
        of key value pairs to update.

        Properties that are not in the request are not modified or removed by this call.

        Server implementations are not required to support namespace properties.
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateNamespacePropertiesRequest'
            examples:
              UpdateAndRemoveProperties:
                $ref: '#/components/examples/UpdateAndRemoveNamespacePropertiesRequest'
      responses:
        200:
          $ref: '#/components/responses/UpdateNamespacePropertiesResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - Namespace not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceNotFound:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        406:
          $ref: '#/components/responses/UnsupportedOperationResponse'
        422:
          description: Unprocessable Entity - A property key was included in both `removals` and `updates`
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                UnprocessableEntityDuplicateKey:
                  $ref: '#/components/examples/UnprocessableEntityDuplicateKey'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/tables:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'

    get:
      tags:
        - Catalog API
      summary: List all table identifiers underneath a given namespace
      description: Return all table identifiers under this namespace
      operationId: listTables
      responses:
        200:
          $ref: '#/components/responses/ListTablesResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - The namespace specified does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceNotFound:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    post:
      tags:
        - Catalog API
      summary: Create a table in the given namespace
      description:
        Create a table or start a create transaction, like atomic CTAS.


        If `stage-create` is false, the table is created immediately.


        If `stage-create` is true, the table is not created, but table metadata is initialized and returned.
        The service should prepare as needed for a commit to the table commit endpoint to complete the create
        transaction. The client uses the returned metadata to begin a transaction. To commit the transaction,
        the client sends all create and subsequent changes to the table commit route. Changes from the table
        create operation include changes like AddSchemaUpdate and SetCurrentSchemaUpdate that set the initial
        table state.
      operationId: createTable
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTableRequest'
      responses:
        200:
          $ref: '#/components/responses/CreateTableResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - The namespace specified does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceNotFound:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        409:
          description: Conflict - The table already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceAlreadyExists:
                  $ref: '#/components/examples/TableAlreadyExistsError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/register:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'

    post:
      tags:
        - Catalog API
      summary: Register a table in the given namespace using given metadata file location
      description:
        Register a table using given metadata file location.

      operationId: registerTable
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterTableRequest'
      responses:
        200:
          $ref: '#/components/responses/LoadTableResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description: Not Found - The namespace specified does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceNotFound:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        409:
          description: Conflict - The table already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                NamespaceAlreadyExists:
                  $ref: '#/components/examples/TableAlreadyExistsError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/tables/{table}:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'
      - $ref: '#/components/parameters/table'

    get:
      tags:
        - Catalog API
      summary: Load a table from the catalog
      operationId: loadTable
      description:
        Load a table from the catalog.


        The response contains both configuration and table metadata. The configuration, if non-empty is used
        as additional configuration for the table that overrides catalog configuration. For example, this
        configuration may change the FileIO implementation to be used for the table.


        The response also contains the table's full metadata, matching the table metadata JSON file.


        The catalog configuration may contain credentials that should be used for subsequent requests for the
        table. The configuration key "token" is used to pass an access token to be used as a bearer token
        for table requests. Otherwise, a token may be passed using a RFC 8693 token type as a configuration
        key. For example, "urn:ietf:params:oauth:token-type:jwt=<JWT-token>".
      parameters:
        - in: query
          name: snapshots
          description:
            The snapshots to return in the body of the metadata. Setting the value to `all` would
            return the full set of snapshots currently valid for the table. Setting the value to
            `refs` would load all snapshots referenced by branches or tags.
          
            Default if no param is provided is `all`.
          required: false
          schema:
            type: string
            enum: [ all, refs ]
      responses:
        200:
          $ref: '#/components/responses/LoadTableResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found - NoSuchTableException, table to load does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToLoadDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    post:
      tags:
        - Catalog API
      summary: Commit updates to a table
      operationId: updateTable
      description:
        Commit updates to a table.


        Commits have two parts, requirements and updates. Requirements are assertions that will be validated
        before attempting to make and commit changes. For example, `assert-ref-snapshot-id` will check that a
        named ref's snapshot ID has a certain value.


        Updates are changes to make to table metadata. For example, after asserting that the current main ref
        is at the expected snapshot, a commit may add a new child snapshot and set the ref to the new
        snapshot id.


        Create table transactions that are started by createTable with `stage-create` set to true are
        committed using this route. Transactions should include all changes to the table, including table
        initialization, like AddSchemaUpdate and SetCurrentSchemaUpdate. The `assert-create` requirement is
        used to ensure that the table was not created concurrently.
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CommitTableRequest'
      responses:
        200:
          $ref: '#/components/responses/CommitTableResponse'
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found - NoSuchTableException, table to load does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToUpdateDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
        409:
          description:
            Conflict - CommitFailedException, one or more requirements failed. The client may retry.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        500:
          description:
            An unknown server-side problem occurred; the commit state is unknown.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              example: {
                "error": {
                  "message": "Internal Server Error",
                  "type": "CommitStateUnknownException",
                  "code": 500
                }
              }
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        502:
          description:
            A gateway or proxy received an invalid response from the upstream server; the commit state is unknown.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              example: {
                "error": {
                  "message": "Invalid response from the upstream server",
                  "type": "CommitStateUnknownException",
                  "code": 502
                }
              }
        504:
          description:
            A server-side gateway timeout occurred; the commit state is unknown.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              example: {
                "error": {
                  "message": "Gateway timed out during commit",
                  "type": "CommitStateUnknownException",
                  "code": 504
                }
              }
        5XX:
          description:
            A server-side problem that might not be addressable on the client.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              example: {
                "error": {
                  "message": "Bad Gateway",
                  "type": "InternalServerError",
                  "code": 502
                }
              }

    delete:
      tags:
        - Catalog API
      summary: Drop a table from the catalog
      operationId: dropTable
      description: Remove a table from the catalog
      parameters:
        - name: purgeRequested
          in: query
          required: false
          description: Whether the user requested to purge the underlying table's data and metadata
          schema:
            type: boolean
            default: false
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found - NoSuchTableException, Table to drop does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToDeleteDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

    head:
      tags:
        - Catalog API
      summary: Check if a table exists
      operationId: tableExists
      description:
        Check if a table exists within a given namespace. The response does not contain a body.
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found - NoSuchTableException, Table not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToLoadDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/tables/rename:
    parameters:
      - $ref: '#/components/parameters/prefix'

    post:
      tags:
        - Catalog API
      summary: Rename a table from its current name to a new name
      description:
        Rename a table from one identifier to another. It's valid to move a table
        across namespaces, but the server implementation is not required to support it.
      operationId: renameTable
      requestBody:
        description: Current table identifier to rename and new table identifier to rename to
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RenameTableRequest'
            examples:
              RenameTableSameNamespace:
                $ref: '#/components/examples/RenameTableSameNamespace'
        required: true
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found
            - NoSuchTableException, Table to rename does not exist
            - NoSuchNamespaceException, The target namespace of the new table identifier does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToRenameDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
                NamespaceToRenameToDoesNotExist:
                  $ref: '#/components/examples/NoSuchNamespaceError'
        406:
          $ref: '#/components/responses/UnsupportedOperationResponse'
        409:
          description: Conflict - The target identifier to rename to already exists as a table or view
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              example:
                $ref: '#/components/examples/TableAlreadyExistsError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/tables/{table}/metrics:
    parameters:
      - $ref: '#/components/parameters/prefix'
      - $ref: '#/components/parameters/namespace'
      - $ref: '#/components/parameters/table'

    post:
      tags:
        - Catalog API
      summary: Send a metrics report to this endpoint to be processed by the backend
      operationId: reportMetrics
      requestBody:
        description: The request containing the metrics report to be sent
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ReportMetricsRequest'
        required: true
      responses:
        204:
          description: Success, no content
        400:
          $ref: '#/components/responses/BadRequestErrorResponse'
        401:
          $ref: '#/components/responses/UnauthorizedResponse'
        403:
          $ref: '#/components/responses/ForbiddenResponse'
        404:
          description:
            Not Found - NoSuchTableException, table to load does not exist
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IcebergErrorResponse'
              examples:
                TableToLoadDoesNotExist:
                  $ref: '#/components/examples/NoSuchTableError'
        419:
          $ref: '#/components/responses/AuthenticationTimeoutResponse'
        503:
          $ref: '#/components/responses/ServiceUnavailableResponse'
        5XX:
          $ref: '#/components/responses/ServerErrorResponse'

  ### /v1/{prefix}/namespaces/{namespace}/tables/{table}/find:
