# Developing APIs with Google Cloud's Apigee API Platform

Examine the OpenAPI specification.

This is the OpenAPI specification for the backend service that will be used in many of the course labs. Let's explore the sections of the OpenAPI spec.

The openapi field specifies the version of the OpenAPI specification. This is an OpenAPI version 3 spec, as indicated by the version number at the top of the file:

openapi: "3.0.0"
The info object provides metadata about the API itself. The version shown is the version of the Retail Backend specification:

info:
  version: 1.0.0
  title: Retail Backend
  description: Retail backend database used for Developing APIs course
  contact:
    name: Google Cloud (Apigee)
    email: apigee@example.org
    url: https://cloud.google.com/apigee
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT
The servers array contains a list of server objects that specify connectivity information for target servers. This specification contains a single backend service, which your API proxy will call:

servers:
- url: "https://gcp-cs-training-01-test.apigee.net/training/db"
  description: Retail backend for Developing APIs course
The tags array adds metadata to tags used in the operations, which are shown below. Tags may be shared by multiple operations, and tags can be used to provide verbose descriptions or links to external documentation.

tags:
  - name: categories
    description: Product Categories
  - name: products
    description: Products
  - name: orders
    description: Orders
  - name: stores
    description: Stores
The paths object holds the relative paths to individual endpoints and their operations. One such path, /categories/{categoryId}, is used to specify a single category. Shown here is a get operation specified to get a category by ID. The get object shows parameters and responses. For operations that contain a request body, like PATCH /products/{productId}, the request body will also be specified.

  /categories/{categoryId}:
    get:
      summary: Get a specific category
      operationId: getCategoryById
      tags:
        - categories
      parameters:
        - name: categoryId
          in: path
          required: true
          description: category id
          schema:
            type: integer
      responses:
        '200':
          description: Selected category
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Category"
        '404':
          description: Category not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        default:
          description: unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
The components object contains reusable objects for different portions of the OpenAPI spec. The securitySchemes component object contains definitions of different types of security schemes used by the operations. This spec defines a single Basic Authentication scheme, which is referenced in the PATCH /products/{productId} operation. The schemas component object contains input and output data types. Shown here is the Category object, which is returned when the GET /categories/{categoryId} operation returns successfully.

components:
  securitySchemes:
    basicAuth:
      type: http
      description: basic auth
      scheme: basic
  schemas:
    Category:
      type: object
      description: product category
      required:
        - color
        - id
        - name
      properties:
        color:
          description: use this color for displaying this category
          type: string
        id:
          description: integer id (used for access)
          type: integer
          format: int32
          minimum: 0
        name:
          description: category name
          type: string
Feel free to explore the specification or the OpenAPI specification documentation.




heck provisioning dashboard
In the Google Cloud Console, navigate to Compute Engine > VM instances.

Click on the External IP for the lab-startup VM.

external IP

If you see a redirect notice page, click the link to the external IP address.

A new browser window will open. Lab startup tasks are shown with their progress.

Create proxies, shared flows, target servers should be complete when you first enter the lab, allowing you to use the Apigee console for tasks like proxy editing.
Create API products, developers, apps, KVMs indicates when the runtime is available and those assets may be saved.
Create KVM data, proxies handle API traffic indicates when the eval environment has been attached to the runtime and the deployed proxies can take runtime traffic.