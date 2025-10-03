
# 🚀 Development Plan with Quarkus + Neo4j (Cypher-DSL + Java Driver) for SCIM

## 1. **Architecture Overview**

* **Framework**: Quarkus (fast startup, cloud-native, supports reactive + Panache-style dev)
* **Persistence**:

  * **Neo4j Java Driver** → execute queries
  * **Cypher-DSL** → type-safe query builder
* **API Layer**: Quarkus RESTEasy Reactive (JAX-RS endpoints)
* **Validation**: Hibernate Validator (Jakarta Bean Validation) + schema-driven validation (from Neo4j `(:Schema)` nodes)
* **Configuration**:

  * `application.properties` for Neo4j connection
  * Support GraalVM native builds for lightweight SCIM service

---

## 2. **Domain Model in Graph (Neo4j)**

We represent SCIM resources as nodes:

* **Schema** (`(:Schema {id, name, description, type})`)
* **ResourceType** (`(:ResourceType {id, name, endpoint, schema})`)
* **Objects**:

  * `(:Object:User {id, userName, active, ...})`
  * `(:Object:Group {id, displayName, members, ...})`
* **Relationships**:

  * `(ResourceType)-[:USES_SCHEMA]->(Schema)`
  * `(User)-[:MEMBER_OF]->(Group)`
  * `(User)-[:HAS_ATTRIBUTE]->(SchemaAttribute)`

This matches SCIM RFC’s flexible schema + extension model.

---

## 3. **Quarkus Project Modules**

### 🔹 API Layer (REST)

* Use **RESTEasy Reactive** (`javax.ws.rs` / `jakarta.ws.rs`)
* Endpoints: `/Users`, `/Groups`, `/Schemas`
* Return SCIM-compliant JSON (with `schemas`, `meta`, etc.)

### 🔹 Service Layer

* Business logic per resource:

  * Validation (check `userName` required, uniqueness)
  * Mapping between SCIM DTOs <-> Neo4j nodes
  * Handling schema extensions dynamically

### 🔹 Persistence Layer

* **Cypher-DSL** to build Cypher queries
* **Neo4j Driver** (`org.neo4j.driver.Driver`) to execute them
* Mapper: `Record → UserDTO / GroupDTO / SchemaDTO`

### 🔹 Schema Registry

* Load SCIM schemas from Neo4j on startup
* Provide **dynamic validation rules** (e.g., mutability, uniqueness)

---

## 4. **SCIM Endpoints with Quarkus**

### 🔹 Users

```java
@Path("/v2/ilm/ds/Users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @Inject UserService userService;

    @POST
    public Response createUser(UserDto user) {
        return Response.ok(userService.create(user)).build();
    }

    @GET
    public Response listUsers(@QueryParam("filter") String filter) {
        return Response.ok(userService.list(filter)).build();
    }

    @GET
    @Path("/{id}")
    public Response getUser(@PathParam("id") String id) {
        return Response.ok(userService.getById(id)).build();
    }

    @PATCH
    @Path("/{id}")
    public Response patchUser(@PathParam("id") String id, PatchRequest patch) {
        return Response.ok(userService.patch(id, patch)).build();
    }

    @DELETE
    @Path("/{id}")
    public Response deleteUser(@PathParam("id") String id) {
        userService.delete(id);
        return Response.noContent().build();
    }
}
```

### 🔹 Groups

Same CRUD as users, plus membership relationship management.

### 🔹 Schemas

* Read schema definitions from Neo4j
* Patch schemas to extend attributes dynamically.

---

## 5. **Using Cypher-DSL in Quarkus**

### Example: Create User

```java
Node u = Cypher.node("User").named("u");

Statement stmt = Cypher.create(u)
    .set(u.property("id").to(Cypher.literalOf(user.getId())))
    .set(u.property("userName").to(Cypher.literalOf(user.getUserName())))
    .set(u.property("active").to(Cypher.literalOf(user.isActive())))
    .returning(u)
    .build();

String cypher = Renderer.getDefaultRenderer().render(stmt);
try (Session session = driver.session()) {
    session.run(cypher);
}
```

### Example: Filter Users (SCIM `filter=active eq true`)

```java
Node u = Cypher.node("User").named("u");
Statement stmt = Cypher.match(u)
    .where(u.property("active").isTrue())
    .returning(u)
    .build();
```

---

## 6. **Validation**

* **Static validation**: Bean Validation for DTOs
* **Dynamic validation**:

  * Fetch schema definition from Neo4j `(:Schema)`
  * Enforce rules:

    * `mutability=readOnly` → ignore in PATCH
    * `required=true` → reject missing values
    * `uniqueness=server` → Cypher query check

---

## 7. **Development Roadmap (Quarkus Sprints)**

### **Phase 1 – Setup**

* [ ] Quarkus project skeleton (`mvn io.quarkus:quarkus-maven-plugin:create`)
* [ ] Integrate Neo4j Java Driver (`quarkus-neo4j` extension or custom CDI bean)
* [ ] Add Cypher-DSL dependency

### **Phase 2 – Schema Engine**

* [ ] Load schemas from Neo4j at startup
* [ ] Implement schema cache
* [ ] Validate User attributes against schema

### **Phase 3 – User Resource**

* [ ] Implement `/Users` CRUD endpoints
* [ ] Implement SCIM filter → Cypher-DSL parser
* [ ] Write integration tests (RestAssured + Testcontainers Neo4j)

### **Phase 4 – Group Resource**

* [ ] Implement `/Groups` CRUD
* [ ] Implement `(User)-[:MEMBER_OF]->(Group)` relationships

### **Phase 5 – Schema Resource**

* [ ] `/Schemas` GET + PATCH
* [ ] Support schema extension nodes

### **Phase 6 – Advanced Features**

* [ ] `/ResourceTypes` endpoint
* [ ] `/Bulk` operations (batch create/update)
* [ ] Pagination & sorting in GET
* [ ] SCIM Search (`filter`, `attributes`, `startIndex`, `count`)

---

## 8. **Testing**

* **Unit tests**: Validate Cypher-DSL query generation
* **Integration tests**: Use QuarkusDevModeTest + Testcontainers Neo4j
* **Conformance tests**: SCIM compliance suite

---

## 9. **Deployment & Ops**

* Quarkus native build for containerized deployment
* Dockerfile with GraalVM native image
* Config via environment (`NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASS`)
* CI/CD pipeline (Maven + Quarkus build plugin)

---

