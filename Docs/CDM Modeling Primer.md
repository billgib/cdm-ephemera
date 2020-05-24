# CDM Modeling Primer
## An unofficial guide to CDM

Bill Gibson, 5/24/2020 <br/>

The Common Data Model (CDM) language provides a way to model data, both the shape or structure of data (schema) and its meaning or purpose (semantics). CDM can define logical models, which express abstract data shapes and semantics, and physical models, which describe how data is structured, formatted, and organized in storage or in some view of the data.

For the official Microsoft CDM docs, see https://docs.microsoft.com/en-us/common-data-model/.  For an unofficial fire-hose look at CDM, read on. 

In CDM, logical and physical data is described in terms of entities and their attributes and relationships to other entities. A logical entity definition, in addition to defining a purely logical perspective of an entity, can also describe how it should be mapped or resolved to a physical entity. The mapping, referred to as resolution guidance, can apply a relational implementation style with foreign key-based relationships between entities, a denormalized flattened implementation, with related entities embedded in line, or a structured implementation, with related data contained in a nested structure.

CDM is based on a type system much like other languages.  Key concepts are sketched and then described below.

![CDM Conceptual Model](https://github.com/billgib/cdm-ephemera/blob/master/Docs/Media/CDMConceptualModel.png)

- **CDM SDK and object model**
  - The CDM SDK provides an object model used to parse, create, process, and validate CDM, including, for example, operations to resolve a logical entity into a physical entity and save it to a CDM folder.
  - The object model also provides tools to map CDM to and from other formats such as the earlier model.json.
  - Given the complexity of CDM, it is strongly recommended to use the the CDM object model in all tools that work directly with CDM.  The code for the CDM object model can be downloaded in several languages from the [CDM GitHub](https://github.com/Microsoft/CDM). 

- **CDM objects (types)**
  - Are manifests, entities, datatypes, traits, and attribute groups, all described below.
  - Are defined by referencing other CDM objects and definitions.

- **CDM definition files**
  - Contain CDM object definitions
  - Are JSON files named \*.cdm.json
  - Manifests are defined one-per file named \*.manifest.cdm.json
  - It's best practice to define entities, one per file, named \<entity\>.cdm.json, although this is not required. Using individual files allows definitions to be worked on, referenced and versioned individually.
  - Other CDM objects are commonly grouped into definition files based on their kind and/or intended use.
  - References to object definitions within files are written as \<pathexpression\>/\<filename\>/\<objectName>, with the path expression typically relative to the model root or corpus location. For example, projects/task.cdm.json/task
  - A CDM definition file can import other CDM definition files to bring CDM objects, defined in that file or imported into that file, into scope so they can be referenced as if defined locally. Most CDM definition files directly or indirectly import the foundations.cdm.json file, which imports primitive CDM datatypes and standard traits.
  - The collection of files required to resolve a definition (typically a manifest) is referred to as the _corpus_.
  - **Import statements** in a CDM definition file use a corpusPath expression to describe the location of the file being imported.  A corpus path expression can take three forms:
    - Contains just the name of the file, indicating that the file is at the same location as the importing file
    - Is prefixed by  "/", indicating a relative path from the model- or corpus root to the file.
    - Is prefixed by an alias and "/", indicating a relative path from the location defined by the alias. For example, the expression, sales:/customer.cdm.json refers to a definition file at the root location specified by the *sales* alias.
  - **Aliases** provide a mechanism to allow the location of imported files to be provided via configuration.  This allows definition files to be persisted at different locations and in different kinds of storage (e.g. GitHub or ADLS, local filesystem, etc.) and allows the user of a model, by changing the configuration of the alias in the config.json file to change the binding. This mechanism can be used, for example, to ensure that referenced content is in an accessible location when CDM folders are deployed.
  - Aliases are defined in a config.json file located with a manifest. Thus different manifests can use different locations for referenced content.
  - While alias values are arbitrary strings, the alias &quot;cdm&quot;, is widely used to refer to the location of the base CDM files, including the CDM foundations and primitives, and the CDM schema These base CDM definitions are mastered in the CDM GitHub repo here [https://github.com/microsoft/CDM/tree/master/schemaDocuments](https://github.com/microsoft/CDM/tree/master/schemaDocuments) and may be referenced from thee directly or may be copied to some other location, such as an ADLS folder or a local file system folder and referenced there.
  - In an alias entry in the config.json file, the *alias type* binds to a storage adapter, values include:
    - "adls" for an ADLS gen2 adapter; the root location can be defined to any ADLS location using a URL specifying the endpoint for the account, container and optionally a folder path
    - "github" for the GitHub adapter, which currently only allows binding to the CDM GitHub repo at the [https://github.com/microsoft/CDM/tree/master/schemaDocuments](https://github.com/microsoft/CDM/tree/master/schemaDocuments) location
    - "local" binds to a local file system adapter; the root location can be any valid file system location
  - If importing files results in multiple objects of the same name, a **Moniker** can be defined on the alias. Prefixing an object name with the moniker in references allows discrimination between same-named objects.<br/><br/>**Caution:** Not all CDM-aware tools support the use of aliases. If not, you may need to make a local copy of referenced definition files and modify import statements.

  Snippet below shows a CDM definition file that imports the CDM Foundations definition using the cdm alias, and a local Person definition file 
  
```
  {
    "$schema": "https://raw.githubusercontent.com/microsoft/CDM/master/schemaDocuments/schema.cdm.json",
    "jsonSchemaSemanticVersion": "1.0.0",
    "imports": [
        {
            "corpusPath": "cdm:/foundations.cdm.json"
        },
        {
            "corpusPath": "Person.cdm.json"
        }
    ],
    "definitions": [
        {...
        } 
    ]
  }
```  


- **CDM folder**
  - Primarily refers to a folder that directly contains a manifest that references, either directly or via sub-manifests, **physical entity definitions with data**. Most commonly, the referenced data is contained in files within the same folder or sub-folders.
  - A folder containing a manifest that references logical entity definitions but does not reference data tends not to be referred to as a CDM folder.
  - A CDM folder **can contain multiple manifest files** that might reference disjoint or overlapping sets of data.
  - While much is made of CDM folders as the container of CDM data, it is the manifest within the folder that is the root object for accessing data and metadata. When processing the manifest, objects and data in the folder that are not referenced by the manifest are 'invisible'. For this reason, it is important when adopting CDM to **ensure that all consumers refer to the manifest and do not process the files in the CDM folder directly**.
  - **One producer, many consumers**. For file-based CDM folders in ADLS, as the data files and CDM metadata must be explicitly managed by the producer - ADLS folders are not a DBMS with transaction support,  - it is a best practice to have only a single producing process for a CDM folder. This minimize the possibilty of contention on the metadata files.  This can be mitigated through the use of sub-manifests, discussed below.     

- **CDM manifest**
  - Defines a group of entities _by reference._ 
  - In CDM, a manifest defines the scope of a CDM 'model'.  While widely used in discussion, the term 'model' is not a formal CDM concept.
  - Defines (typically) either a group of logical entity definitions – a logical model, or a group of physical entity definitions plus data.
  - Contains one or more entity declarations that each references an entity definition in a CDM definition file. CDM definition files are typically located within the parent folder of the manifest or within sub-folders or at some other location referenced via an alias.
  - If a manifest describes a CDM folder containing data, each entity declaration will also contain partitions or partition patterns that define where the data is located and how it is formatted, e.g. CSV or Parquet.
  - A manifest can contain sub-manifests.
    - A sub-manifest is a reference to a separate manifest (file) that includes the content of that manifest as part of the parent manifest.
    - Manifests that are included as sub-manifests can include other sub-manifests.
    - Sub-manifests provide a way to separate entities or groups of entities and allow these groups to be reused if desired.
    - Using sub-manifests can reduce the risk of contention on the root manifest file if large numbers of entities are included and may be updated in parallel, requiring the manifest to be updated. 
    - A common pattern is to use a sub-manifest per entity in a CDM folder.  This is the default pattern used by mapping dataflows in ADF and Synapse, for example.
    - With care, sub-manifests could be used by different producers contributing distinct sets of entities into a larger CDM folder.
  - When a manifest is loaded and resolved by the CDM object model, it also loads the set of CDM definition files containing all directly and indirectly referenced objects. The body of related CDM definition files is referred to as a corpus. The corpus root is defined typically by a manifest location, and all other files and objects can be referenced relative to that corpus root or by referencing an alias and a relative path expression.
  - If a CDM folder references shared definitions these can be in higher level folders, in which case, the corpus root must be set to the highest folder containing referenced definitions.

  The snippet below shows a manifest with a single entity declaration for the Person entity, defined in the Person.cdm.json file located in the same folder as the manifest.

```
{
    "$schema": "https://raw.githubusercontent.com/microsoft/CDM/master/schemaDocuments/schema.manifest.cdm.json",
    "jsonSchemaSemanticVersion": "1.0.0",
    "imports": [
        {
            "corpusPath": "cdm:/foundations.cdm.json"
        }
    ],
    "manifestName": "Contacts",
    "explanation": "A logical model of contacts",
    "entities": [
        {
            "type": "LocalEntity",
            "entityName": "Person",
            "entityPath": "Person.cdm.json/Person"
        }
    ]
}
```

- **Partitions and partition patterns**
  - An entity declaration can contain a list of partitions and/or partition patterns that identify dataset(s) that are defined by the referenced physical entity definition.
  - A _partition_ identifies a specific dataset – typically a file – that contains all or part of the entity's data.
  - A _partition pattern_ identifies datasets implicitly by defining a path and/or name pattern and a format that applies to all partitions that conform to the pattern. The pattern can be defined using either a regex or glob wildcard expression. A glob wildcard pattern tends to be more easily understood, although is less expressive than regex.
  - The CDM object model provides a function to resolve a partition pattern and return a list of partitions that conform to that pattern.
  - Individual partitions and partition patterns the format of the referenced dataset(s), such as csv or parquet.
  
  **Note:** for any given entity, there is no requirement that all partitions/partition patterns use the same format; it is OK to mix csv and parquet files for example. Regardless of the formats used, the data is combined; **different formats are not interpreted as alternative versions of the same data**. 
  
  CDM partitions and partition patterns might in future reference other data stores and formats, such as a table in SQL or a folder in Delta Lake.

- **Entity versioning**
  - By default, datasets referenced by partitions or partition patterns included in an entity declaration are defined by the referenced entity definition.
  - If the schema of an entity changes over time, older partitions can reference an earlier version of the physical entity definition. If care is taken to ensure that schema changes are additive and new attributes are introduced are nullable or have a default value, this mechanism allows clients to be able to read new and old data.
  - A version number trait can be associated with an entity and a numbering scheme used to indicate whether an entity version is compatible with earlier versions. Such usage and schemes are not enforced by the CDM object model. The onus is on the producer to correctly mark versions and on the client to recognize the presence of different versions and to process the metadata and react accordingly.

  *Versioning is an advanced CDM scenario that may not be supported by all CDM-aware tools. Verify before using.*

- **Entity definitions** are of two kinds: logical and physical.
  - A **logical entity definition** :
    - Defines entity shape and semantics only.
    - Is unresolved; all references to other CDM objects are explicit.
    - Is described in terms of attributes. Relationships are expressed using entity attributes.
    - Can extend another entity and add further attributes.
    - Is defined in a CDM definition file that explicitly imports definition files required to directly or indirectly bring any referenced CDM objects into scope, including entities being extended, entities and datatypes referenced by attributes, including CDM primitive datatypes, reusable attribute groups, and traits applied to the entity, attributes or datatypes.
    - May influence how it is mapped to a physical entity definition using resolution guidance directives.  See *Resolution Guidance* below

    Including resolution guidance blurs the distinction between logical and physical entities. To avoid polluting a logical entity with physical layout information, consider putting resolution guidance in an extended entity type. In this way multiple physical definitions can be defined from the same logical definition or using CDM object model resolution directives to guide the resolution.

The snippet below shows the definition for two entites, Entity and Person.  Entity is a base entity extended by Person.  When resolved, the physical Person entity will acquire the attributes of Entity. 

```
    "definitions": [
        {
            "entityName": "Entity",
            "description": "A base entity type defining common attributes used by other entities",
            "hasAttributes": [
                {
                    "purpose": "hasA",
                    "dataType": "integer",
                    "name": "identifier",
                    "description": "The identifier of the entity"
                },
                {
                    "purpose": "hasA",
                    "dataType": "dateTime",
                    "name": "createdTime",
                    "description": "The UTC time this entity was created"
                }
            ]
        },        
        {
            "entityName": "Person",
            "extendsEntity": "Entity",
            "description": "An individual",
            "hasAttributes": [
                {
                    "purpose": "hasA",
                    "dataType": "string",
                    "name": "firstName",
                    "description": "The person's first name",
                    "maximumLength": 100
                },
                {
                    "purpose": "hasA",
                    "dataType": "string",
                    "name": "lastName",
                    "description": "The person's last name",
                    "maximumLength": 100
                },
                {
                    "purpose": "hasA",
                    "dataType": "date",
                    "name": "birthDate",
                    "description": "The person's date of birth"
                }
            ]
        }
    ]
```

  - A **physical entity definition** :
    - Is a fully resolved entity definition, typically created by using the CDM object model to resolve a logical entity definition and saving the resulting resolved entity definition. Resolving a logical entity creates a new physical entity that:
      - Includes all traits of the logical entity
      - Includes all the attributes of the logical entity and their traits
      - Includes any attributes inherited by the logical entity inline,
      - Includes all attributes from any attribute groups inline,
      - Applies all resolution guidance directives, for example, including foreign key attributes and other selected attributes from related entity types in-line if required.
    - Is largely a self-contained entity definition that can be processed without further resolution.

The resolution process currently adds a single import statement to a physical entity definition to import the file containing the logical entity definition from which it was resolved. This does mean that the **logical entity definition must be available at 'run time'**. This requirement is being reviewed.

- **Attributes**
  - An attribute defines a slot on an entity that holds data about an instance of the entity. Examples  include, firstName, lastName, and birthdate on the Person entity snippet above.
  - Attribute properties include name, description, display name
  - Attributes can be of a **datatype** , which might be a CDM primitive type like string or a custom datatype, like emailAddress, which might be defined as a string with additional formatting constraints defined. Properties include, nullable, maximum length, minimum and maximum value, purpose (such as identifier).
  - Attributes can be of an **entity type** , for example, Business.owner is of type Person. This defines a logical relationship between the Business and Person entities. Properties include min and maximum cardinality of the related entity.
  - Resolution guidance can be provided on entity attributes on logical entities that indicate how the relationship should be manifested in the resolved physical entity.

- **Relationships**
  - Relationships in a purely logical model are defined by an entity attribute.
  - CDM currently doesn't require you create entity attributes at both ends of a relationship, and if you do, it doesn't yet support the notion of binding a pair of entity attributes as the inverse of each other.
  - Related entity types can be implemented in different ways. Resolution guidance on the entity can be used to indicate whether you want to use a relational implementation with a foreign key, or to embed a denormalized copy of a related entity, or support the notion of a structured or nested hierarchy of objects, which suits JSON or nested Parquet formats.
  - If you choose to use a relational approach, you can then separately represent the connected notion of a relationship (with _from_ and _to_ entities) as a relationship object in the manifest.  It is possible to generate the relationships in the manifest from the entities and their resolution guidance.

- **Traits**
  - While entities and attributes describe the shape of data, traits provide an extensible way to classify CDM definitions with semantics or meaning. Traits are metadata that can be applied to a CDM object and which define characteristics that apply to all instances of that object.
  - Traits are named using a hierarchical namespace-like convention, e.g. is.dataFormat.Integer, is.partition.format.parquet, means.location.address.city, means.measurement.duration.minutes, means.identity.person.lastName, etc.
  - Traits may be simple tags or take parameters, such as precision and scale values on the trait is.dataformat.decimal.
  - There is base set of traits that ships with CDM and further traits can be defined to define an ontology for any domain of interest.

- **DataTypes**
  - Datatypes define the logical type of a single datum.
  - CDM defines a set of standard datatypes, such as string, date, time, etc. which can be extended by custom datatypes.
  - Extending the primitive datatypes provides a way to define reusable semantic type definitions for attributes, like emailAddress. Datatypes can be tagged with traits to convey their meaning and to express constraints, such as maximum and minimum values or sizes, precision and range, default values.
  - When resolving an entity, logical Datatypes on the attributes are resolved to physical DataFormats with a set of data format traits that describe the semantics at the datatype level.

- **Attribute groups**
  - Attribute groups provide a way to group together attributes that are commonly used together but are not considered a datatype or entity.
  - AttributeGroups can be included in an entity definition like individual attributes.

- **Entity resolution guidance and resolution**
  - A logical entity can include resolution guidance instructions that influence how it is resolved. Depending on the options chosen, mappings to different physical shapes are possible, particularly influencing how entity attributes are resolved, whether to a relational style (with FKs), denormalized/embedded with attributes from the realted entity inline, or nested/hierarchical suitable for formats like nested Parquet.
  - Using extension, it's possible to create alternate physical entities from the same base logical entity.
  - Resolution is normally performed by the CDM object model. Resolution creates a physical entity (see above). Resolved entity types could be authored by hand but it's not recommended.  
  - A CDM folder will normally include the resolved entity and may also include the logical entity from which it was resolved, or this may be referenced at a remote location, particularly where the definition is part of a shared model.
