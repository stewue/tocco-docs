.. _user-types:

Custom user types
=================

Per default Hibernate can map all primitive types (and its wrapper classes) as well as references to other entities.
For all other classes that need to be mapped to the database an :java-hibernate:`UserType <org/hibernate/usertype/UserType>`
must be implemented (for immutable types the base class :abbr:`ImmutableUserType (ch.tocco.nice2.persist.hibernate.usertype.ImmutableUserType)`
can be used). The user type contains the logic how a specific object should be read from the :java:`ResultSet <java/sql/ResultSet>`
and written to the :java:`PreparedStatement <java/sql/PreparedStatement>`.

For example the :abbr:`Login (ch.tocco.nice2.types.Login)` class is mapped with a custom user type (:abbr:`LoginUserType (ch.tocco.nice2.persist.hibernate.usertype.LoginUserType)`).

*There are two ways to register a new user type:*

It can be registered with a bootstrap contribution (see :ref:`bootstrap`). This way should be used if
there is a distinct object which should be mapped (for example :abbr:`Login (ch.tocco.nice2.types.Login)`):

.. code-block:: java

    classLoaderService.addContribution(TypeContributor.class, ((typeContributions, serviceRegistry) -> {
        typeContributions.contributeType(new BinaryUserType(binaryAccessProvider, binaryHashingService), Binary.class.getName());
        typeContributions.contributeType(new LoginUserType(), Login.class.getName());
        typeContributions.contributeType(new UuidToStringUserType(), UUID.class.getName());
    }));

The alternative is to bind the user type to a field type (like ``phone``) using a contribution:

.. parsed-literal::

    <contribution configuration-id="nice2.persist.core.UserTypeContributions">
        <contribution type="phone" implementation="service:PhoneUserType"/>
    </contribution>

.. note::

    User types are going to be deprecated in Hibernate 6+ and some refactoring will be required.
    One attempt to replace them with :java-javax:`AttributeConverter <javax/persistence/AttributeConverter>`
    was not successful due to classloading issues with hivemind.

Simple user types
-----------------

Most user types are relatively simple and only map between a string or number and an object:

    * :abbr:`EncodedPasswordUserType (ch.tocco.nice2.security.spi.auth.hibernate.EncodedPasswordUserType)` maps
      instances of :abbr:`EncodedPassword (ch.tocco.nice2.types.spi.password.EncodedPassword)` from/to a string.
    * :abbr:`LoginUserType (ch.tocco.nice2.persist.hibernate.usertype.LoginUserType)` maps
      instances of :abbr:`Login (ch.tocco.nice2.types.Login)` from/to a string.
    * :abbr:`UuidToStringUserType (ch.tocco.nice2.persist.hibernate.usertype.UuidToStringUserType)` maps
      instances of :java:`UUID <java/util/UUID>` from/to a string.
    * :abbr:`GeolocationTypesContribution (ch.tocco.nice2.optional.geolocation.impl.type.GeolocationTypesContribution)` contains
      user types that support :abbr:`Latitude (ch.tocco.nice2.types.spi.geolocation.Latitude)` and :abbr:`Longitude (ch.tocco.nice2.types.spi.geolocation.Longitude)` objects.

``phone`` type
--------------

The :abbr:`PhoneUserType (ch.tocco.nice2.entityoperation.impl.phone.PhoneUserType)` is applied for all field
of the virtual ``phone`` type.
This user type does not convert between different objects, but formats the phone number using the
:abbr:`PhoneFormatter (ch.tocco.nice2.toolbox.phone.PhoneFormatter)` whenever a ``phone`` value
is written to the database.

``html`` type
-------------

Like the :abbr:`PhoneUserType (ch.tocco.nice2.entityoperation.impl.phone.PhoneUserType)`, the
:abbr:`HtmlUserType (ch.tocco.nice2.persist.hibernate.usertype.HtmlUserType)` does not convert between different objects
but does some string formatting for ``html`` fields.

The formatting behaviour can be contributed using a :abbr:`HtmlUserTypeExtension (ch.tocco.nice2.persist.hibernate.usertype.HtmlUserTypeExtension)`.
Currently there is only the :abbr:`PreserveFreemarkerOperatorsHtmlUserTypeExtension (ch.tocco.nice2.templating.impl.freemarker.usertype.PreserveFreemarkerOperatorsHtmlUserTypeExtension)`
which handles escaping in freemarker expressions.

``binary`` type
---------------

The :abbr:`BinaryUserType (ch.tocco.nice2.persist.hibernate.usertype.BinaryUserType)` handles the :abbr:`Binary (ch.tocco.nice2.persist.entity.Binary)`
class. The column of a ``binary`` field contains the hash code of the binary and references the ``_nice_binary`` table.

In addition to the mapping of the hash code this user type also calls the configured :abbr:`BinaryAccessProvider (ch.tocco.nice2.persist.spi.binary.BinaryAccessProvider)`
which stores the binary data if necessary.

User types are also used to map query parameters. If a :abbr:`Binary (ch.tocco.nice2.persist.entity.Binary)` object is
used as a query parameter, it should obviously not be attempted to write it to the binary data store!
Therefore the binary content is only saved if ``Binary#mayBeStored()`` returns true.
If a hash code is used as a query parameter for a binary field, the string is converted to a :abbr:`BinaryQueryParameter (ch.tocco.nice2.persist.hibernate.legacy.BinaryQueryParameter)`
by the :abbr:`StringToBinaryParameterConverter (ch.tocco.nice2.persist.hibernate.legacy.StringToBinaryParameterConverter)`.
``BinaryQueryParameter#mayBeStored()`` returns false so it can safely be used in queries.

See chapter :ref:`large_objects` for more details about large objects.

