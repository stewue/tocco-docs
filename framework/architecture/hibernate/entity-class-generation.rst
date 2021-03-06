Entity class generation
=======================

Introduction
------------
All entities in nice2 are currently defined by \*.xml files, which are then parsed
into an :abbr:`EntityModel (ch.tocco.nice2.persist.model.EntityModel)` instance.

There are two different :java-hibernate:`EntityMode <org/hibernate/EntityMode>` in hibernate:

- POJO (a different class per entity)
- MAP (entities based on maps)

As the two modes cannot be mixed and we would like to be able to use typed entities (instead of just the
:abbr:`Entity (ch.tocco.nice2.persist.entity.Entity)` interface) in the future we need to dynamically generate classes for the entity models.

The `javassist <https://www.javassist.org/>`_ library is used to generate the classes. The same
library is used by  Hibernate 5.2.x itself to generate proxy classes. `Bytebuddy <https://bytebuddy.net/>`_ would
be a more modern alternative, but the newest version is much `slower <https://stackoverflow.com/questions/45456076/bytebuddy-performance-in-hibernate>`_ than javassist.

Class generation
----------------
The classes are generated during startup by the :abbr:`JavassistEntityPojoFactory (ch.tocco.nice2.persist.hibernate.pojo.ch.tocco.nice2.persist.hibernate.pojo.JavassistEntityPojoFactory)`.
At first an 'empty' class is generated for each entity model, so that they can already be referenced by other classes.

All generated entity classes inherit from the same base class (:abbr:`AbstractPojoEntity (ch.tocco.nice2.persist.hibernate.pojo.AbstractPojoEntity)`),
which implements all of the logic required by the :abbr:`Entity (ch.tocco.nice2.persist.entity.Entity)`
interface.

The generated classes contain the following:

* a private field of the corresponding type for each field and relation
* a getter and setter for each field
* JPA annotations, which define the hibernate data model, on the fields

If the annotations are placed on the fields (instead of the getters), hibernate reads and writes from the fields
directly, without using the setters and getters.

This has the advantage that we can use the entity interceptors (e.g. security) in the getters and setters, which
is necessary if we want to be able to use the created classes directly (instead of using the ``Entity`` interface)
in the future. It is also required for the 'script listener' functionality, which also uses the getters and setters directly.

Transient entities
^^^^^^^^^^^^^^^^^^

There are so called 'session-only' entities in nice2 which are not mapped to the database (used only for data binding and the like).
A different base class (:abbr:`AbstractSessionOnlyEntity (ch.tocco.nice2.persist.hibernate.pojo.AbstractSessionOnlyEntity)`)
is used for those entities and no JPA annotations are added.
They are basically normal java beans that implement the :abbr:`Entity (ch.tocco.nice2.persist.entity.Entity)` interface.
If such an entity is used in an association with a normal entity, no JPA annotations may be used on both sides, only
a normal field (or collection) is created.

Entities
^^^^^^^^

Apart from the usual :java-javax:`Entity <javax/persistence/Entity>` annotation, the database table name is
explicitly defined with the :java-javax:`Table <javax/persistence/Table>` annotation (we need to use the same
naming strategy to be compatible with existing databases).
A custom :java-hibernate:`EntityPersister <org/hibernate/persister/entity/EntityPersister>` is defined as well (see
:ref:`persister` for details).

Fields
^^^^^^

All fields are annotated with the :java-javax:`Column <javax/persistence/Column>` annotation to define the
column name of this field (we need to use the same naming strategy to be compatible with existing databases).

**Primary Key**

The primary key must be annotated with :java-javax:`Id <javax/persistence/Id>`. If the key value is generated
by the database the annotation :java-javax:`GeneratedValue <javax/persistence/GeneratedValue>` is required as well.
For autoincrement columns, the correct strategy is ``IDENTITY``.

**Version**

Fields of type version are annotated with :java-javax:`Version <javax/persistence/Version>`, which enables optimistic
locking for this entity.

**Text fields**

The ``text`` datatype is a :java:`String <java/lang/String>` that should be saved into a column with datatype
``text``. To achieve this we add the :java-javax:`Lob <javax/persistence/Lob>` annotation to the property.

.. note::
    In Hibernate 5.2.10 a String property annotated with :java-javax:`Lob <javax/persistence/Lob>` was automatically
    mapped to a ``text`` column in PostgreSQL.

    However the behaviour changed in version 5.2.11 (see the `migration guide <https://github.com/hibernate/hibernate-orm/wiki/Migration-Guide---5.2>`_).
    To be compatible with existing databases, we need the behaviour of 5.2.10. In order to accomplish this, a custom
    :java-hibernate:`ClobTypeDescriptor <org/hibernate/type/descriptor/sql/ClobTypeDescriptor>` is registered in the :abbr:`ToccoPostgreSQLDialect (ch.tocco.nice2.persist.hibernate.dialect.ToccoPostgreSQLDialect)`
    which restores the behaviour of 5.2.10.

**Counter fields**

The ``counter`` datatype is a numeric type whose value is automatically generated. The value is incremented for every new entity instance.
The counter values are managed in the ``nice_counter`` table.

Counter fields are annotated with :abbr:`Counter (ch.tocco.nice2.persist.hibernate.pojo.generator.Counter)`, which configures
the :abbr:`CounterGeneration (ch.tocco.nice2.persist.hibernate.pojo.generator.CounterGeneration)` value generator. This
generator is only applied whenever a new entity is inserted (not when an entity is updated).

If the value of a counter field is manually set in the transaction it will not be overwritten.

At first, the counter entity (for the relevant entity type, field and business unit) is fetched from the database
using the ``PESSIMISTIC_WRITE`` lock mode.
The counter value is then updated using a `stateless session <https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#_statelesssession>`_ to make sure that
database is updated immediately. This is necessary if the same counter is used multiple times in the same transaction.
It is important that the connection of the current session is also used in the stateless session to make sure that they use
the same database transaction.

.. note::
    It would probably make sense to use a database ``sequence`` for this purpose in the future.

**Custom user types**

Custom user types are mapped using the :java-hibernate:`Type <org/hibernate/annotations/Type>` annotation.
See the chapter :ref:`user-types` for more details.

**Other fields**

The ``nullable``, ``unique`` and if applicable ``precision`` and ``scale`` properties are set on the :java-javax:`Column <javax/persistence/Column>` annotation.
These properties are only used for schema generation in test cases (databases are setup by liquibase), not for
validation!
The type ``decimal`` (without precision and scale) is handled specially, because Hibernate would use a default
precision and scale, but in this case we want to use the column type ``decimal`` without any precision or scale.

.. _generated-fields-annotations:

Generated fields
^^^^^^^^^^^^^^^^

It is possible to define custom data types whose values are automatically set when an entity is saved or updated.
These fields are annotated either with the :abbr:`AlwaysGeneratedValue (ch.tocco.nice2.persist.hibernate.pojo.generator.AlwaysGeneratedValue)`
for fields which should be updated on create and update or the :abbr:`InsertGeneratedValue (ch.tocco.nice2.persist.hibernate.pojo.generator.InsertGeneratedValue)`
for fields which should only be updated when the entity is created.

See :ref:`generated-values`.

Associations
^^^^^^^^^^^^

Associations (relations) are annotated with one of the following JPA annotations (depending on the type):

- :java-javax:`OneToMany <javax/persistence/OneToMany>`
- :java-javax:`ManyToOne <javax/persistence/ManyToOne>`
- :java-javax:`ManyToMany <javax/persistence/ManyToMany>`

So far all associations are bi-directional (even if this does not always make sense).
In a ManyToOne/OneToMany association, the ManyToOne side is always the owning side. In a ManyToMany association,
the owning side needs to be explicitly specified (with the :java-javax:`JoinTable <javax/persistence/JoinTable>`
annotation).
The owning side is responsible for persisting the relationship - if a change is only done on the inverse side of
an association, it will not be persisted! For example in a ManyToMany association, entities must always be added
and removed from the owning side, otherwise the mapping table won't be updated.

For collections a :java:`LinkedHashSet <java/util/LinkedHashSet>` is used, because we want :java:`LinkedHashSet <java/util/Set>` semantics
(no duplicates), but need to iterate over the elements in the same order as they were inserted (to support sorting by the database).

All associations (including ManyToOne) are configured to be loaded lazily by specifying the :java-javax:`FetchType <javax/persistence/FetchType>`
on the annotation. Per default only to many associations are loaded lazily, that's why we need to explicitly configure
it for to one associations.

When a collection has been initialized it cannot be reloaded from the database (unless the entire object is reloaded).
However when a  :abbr:`Relation (ch.tocco.nice2.persist.entity.Relation)` is resolved, the data should always be
loaded from the database (because this was the behaviour of the old persistence implementation).
To support this behaviour we use a custom collection type (:java-hibernate:`CollectionType <org/hibernate/annotations/CollectionType>`).

See :ref:`collections` chapter for more details.

A custom :java-hibernate:`CollectionPersister <org/hibernate/persister/collection/CollectionPersister>` is also configured (see
:ref:`persister` for details).

Class loading
-------------

The :abbr:`ClassUtils (ch.tocco.nice2.persist.hibernate.session.ClassUtils)` can be used to load the generated classes
by name.
The classes are retrieved from the hibernate :java-hibernate:`Metamodel <org/hibernate/Metamodel>`. The reason for this is that
those classes are generated during the initialization of Hibernate and getting them from the Metamodel ensures
that the classes have been properly initialized (in contrast to loading them directly from the class loader).

