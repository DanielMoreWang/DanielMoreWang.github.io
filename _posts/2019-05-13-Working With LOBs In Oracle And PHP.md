---
layout:     post
title:      Working With LOBs In Oracle And PHP
subtitle:   How to working oracle Lobs with php
date:       2019-05-13
author:     zakun
header-img: img/big_banner/post-bg-keybord.jpg
catalog: true
tags:
    - oracle
    - php
    - clob
---
The Oracle + PHP Cookbook
===

Working with LOBs in Oracle and PHP
---

<a href="https://www.oracle.com/technetwork/articles/fuecks-lobs-095315.html" target="_blank">原文链接</a>

### Hitting the 4,000-byte limit? Enter LOBs...

Downloads for this article:
   * Oracle Database 10g
   * Zend Core for Oracle
   * Apache HTTP Server 1.3 and later
Working with Oracle types like VARCHAR2 is fine, but what if you need to be able to store more than its 4,000-byte limit in one go? For this task, you need one of Oracle's Long Object (LOB) types, which in turn requires that you learn how to use the PHP API for working with LOBs. That in itself can be daunting for those unfamiliar with it.

In this "Oracle+PHP Cookbook" HowTo, you will learn the available LOB types and issues related to them, then explore examples of common LOB operations in PHP.

Long Objects in Oracle

Oracle provides the following LOB types:

* BLOB, used to store binary data
* CLOB, used to store character data using the database character set encoding
* NCLOB, used to store Unicode character data using the national character set. Note that NCLOBs are, currently, not supported by the PHP OCI8 extension, which you'll be using here.
* BFILE, used to reference external files under the operating system's filesystem

A further subcategory of LOB is the temporary LOB, which can be either a BLOB, CLOB, or NCLOB but is stored in the temporary tablespace until you free it.
Note that older versions of Oracle provided the LONG and LONG RAW types for character and binary data, respectively. With Oracle9i these were deprecated in favor of LOBs.

LOB storage. For the BLOB, CLOB, and NCLOB types, Oracle Database 10g is capable of storing up to 128TB in a single value, depending on your database block size and the "chunk" setting, defined for the LOB.

A LOB itself comprises two elements: the LOB content and the LOB locator, which is a "pointer" to the LOB content. This separation is required to allow Oracle to store and manage LOBs efficiently and it is reflected in the PHP APIs you use to INSERT, UPDATE, and SELECT LOBs (see below).

For the internal LOB types (i.e. not BFILEs) Oracle will store the content of the LOB "in-line" in the table, with the rest of the row, if the size of the LOB is less than 4KB. LOBs larger than 4KB are stored "out-of-line," by default in the table's tablespace. This approach allows small LOBs to be retrieved quickly while, for large LOBs, access times will be slower but overall performance, when scanning the table, is preserved.

There are further options for LOB storage and access—such as memory caching and buffering—which may improve performance, depending on the specifics of your application. For further information see the LOB Performance Guidelines and the Oracle Database Application Developer's Guide - Large Objects in the Oracle documentation.

Restrictions on LOBs. A number of restrictions apply to the use of LOB types, the most important being their use in SQL statements. You cannot use a LOB type in any of the following queries.

	SELECT DISTINCT <lob_type>
	ORDER BY <lob_type>
	GROUP BY <lob_col>

It is also illegal to use a LOB type column for table joins, UNION, INTERSECTION, and MINUS statements.
Further restrictions apply to other aspects of the use of LOBs, such as you cannot use LOB as a primary key column. Again, see [Oracle Database Application Developer's Guide](http://download.oracle.com/docs/cd/B19306_01/appdev.102/b14249/toc.htm) - Large Objects for details.

### CLOBs and Character Sets

The default character set for your database is defined by the parameter NLS_CHARACTERSET and text placed in a CLOB is expected to be encoded using this character set. Use this SQL to determine your databases character set encoding:

	SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET'

Given lack of support for NCLOBs in PHP, you may want to consider using Unicode encoding as the database character, such as UTF-8, which can be done (given sufficient privileges) using the statement:
ALTER DATABASE CHARACTER SET UTF8
Note: do not attempt this without understanding the impact, especially if you have existing data or application code using a different character set. See the Oracle Globalization Support Guide and An Overview on Globalizing Oracle PHP Applications for more information.
Working with LOBs

The discussion here will focus on PHP's OCI8 extension. It's also worth noting that Oracle provides the DBMS_LOB package, containing parallel procedures and functions for working with LOBs using PL/SQL.

The PHP OCI8 extension registers a PHP class called "OCI-Lob" in the global PHP namespace. When you execute a SELECT statement, for example, where one of columns is a LOB type, PHP will bind this automatically to an OCI-Lob object instance. Once you have a reference to an OCI-Lob object, you can then call methods like load() and save() to access or modify the contents of the LOB.

The available OCI-Lob methods will depend on your PHP version, PHP5 in particular having gained methods like read() , seek() , and append() . The PHP Manual is a little unclear, in this case, on the version numbers so if in doubt you can verify using the following script.

	<?php
	foreach (get_class_methods('OCI-Lob') as $method ) {
		print "OCI-Lob::$method()\n";
	}
	?>

On my system, running PHP 5.0.5, I get the following list of methods:

OCI-Lob::load()
OCI-Lob::tell()
OCI-Lob::truncate()
OCI-Lob::erase()
OCI-Lob::flush()
OCI-Lob::setbuffering()
OCI-Lob::getbuffering()
OCI-Lob::rewind()
OCI-Lob::read()
OCI-Lob::eof()
OCI-Lob::seek()
OCI-Lob::write()
OCI-Lob::append()
OCI-Lob::size()
OCI-Lob::writetofile()
OCI-Lob::writetemporary()
OCI-Lob::close()
OCI-Lob::save()
OCI-Lob::savefile()
OCI-Lob::free()

In practice, the PHP 4.x OCI8 extension supports reading or writing of complete LOBs only, which is the most common use case in a Web application. PHP5 extends this to allow reading and writing of "chunks" of a LOB as well as supporting LOB buffering with the methods setBuffering() and getBuffering() . PHP5 also provides the stand-alone functions oci_lob_is_equal() and oci_lob_copy() .
The examples here will use the new PHP5 OCI function names (e.g. oci_parse instead of OCIParse ). Examples use the following sequence and table:

	CREATE SEQUENCE mylobs_id_seq
		NOMINVALUE
		NOMAXVALUE
		NOCYCLE
		CACHE 20
		NOORDER
	INCREMENT BY 1;

	CREATE TABLE mylobs (
		id NUMBER PRIMARY KEY,
		mylob CLOB
	)

Note that most of the examples here use CLOBs but the same logic can be applied almost exactly to BLOBs as well.
Inserting a LOB

To INSERT an internal LOB, you first need to initialize the LOB using the respective Oracle EMPTY_BLOB or EMPTY_CLOB functions—you cannot update a LOB that contains a NULL value.

Once initialized, you then bind the column to a PHP OCI-Lob object and update the LOB content via the object's save() method.

The following script provides an example, returning the LOB type from the INSERT query:

	<?php
	// connect to DB etc...

	$sql = "INSERT INTO
			mylobs
			  (
				id,
				mylob
			  )
		   VALUES
			  (
				mylobs_id_seq.NEXTVAL,
				--Initialize as an empty CLOB
				EMPTY_CLOB()
			  )
		   RETURNING
			  --Return the LOB locator
			  mylob INTO :mylob_loc";

	$stmt = oci_parse($conn, $sql);

	// Creates an "empty" OCI-Lob object to bind to the locator
	$myLOB = oci_new_descriptor($conn, OCI_D_LOB);

	// Bind the returned Oracle LOB locator to the PHP LOB object
	oci_bind_by_name($stmt, ":mylob_loc", $myLOB, -1, OCI_B_CLOB);

	// Execute the statement using , OCI_DEFAULT - as a transaction
	oci_execute($stmt, OCI_DEFAULT)
		or die ("Unable to execute query\n");

	// Now save a value to the LOB
	if ( !$myLOB->save('INSERT: '.date('H:i:s',time())) ) {

		// On error, rollback the transaction
		oci_rollback($conn);

	} else {

		// On success, commit the transaction
		oci_commit($conn);

	}

	// Free resources
	oci_free_statement($stmt);
	$myLOB->free();


	// disconnect from DB etc.
	?>

Notice how this example uses a transaction, instructing oci_execute with OCI_DEFAULT constant to wait for an oci_commit or an oci_rollback . This is important as I have two stages taking place in the INSERT—first create the row and second update the LOB.
Note that if I was working with a BLOB type, the only change needed (assuming a BLOB column) is to the oci_bind_by_name call:

oci_bind_by_name($stmt, ":mylob_loc", $myLOB, -1, OCI_B_BLOB);
Alternatively, you can bind a string directly without specifying a LOB type;

	<?php
	// etc.

	$sql = "INSERT INTO
			  mylobs
			  (
				id,
				mylob
			  )
			VALUES
			  (
				mylobs_id_seq.NEXTVAL,
				:string
			  )
	";

	$stmt = oci_parse($conn, $sql);

	$string = 'INSERT: '.date('H:i:s',time());

	oci_bind_by_name($stmt, ':string', $string);

	oci_execute($stmt)
		or die ("Unable to execute query\n");

	// etc.
	?>

This approach simplifies the code significantly and is suitable when the data your want to write to the LOB is relatively small. In contrast, if you wished to stream the contents of the large file into a LOB, you might loop through the contents of the file calling write() and flush() on the PHP LOB object to write smaller chunks rather than having the entire file held in memory at a single instance.
Selecting a LOB

When a SELECT query contains a LOB column, PHP will automatically bind the column to an OCI-Lob object. For example:

	<?php
	// etc.

	$sql = "SELECT
			  *
			FROM
			  mylobs
			ORDER BY
			  Id
	";

	$stmt = oci_parse($conn, $sql);

	oci_execute($stmt)
		or die ("Unable to execute query\n");

	while ( $row = oci_fetch_assoc($stmt) ) {
		print "ID: {$row['ID']}, ";

		// Call the load() method to get the contents of the LOB
		print $row['MYLOB']->load()."\n";
	}

	// etc.
	?>

This can be further simplified using the OCI_RETURN_LOBS constant, used with oci_fetch_array(), instructing it to replace LOB objects with their values:
while ( $row = oci_fetch_array($stmt, OCI_ASSOC+OCI_RETURN_LOBS) ) {
    print "ID: {$row['ID']}, {$row['MYLOB']}\n";
}
Updating a LOB
To UPDATE a LOB, it's also possible to use the "RETURNING" command in the SQL, as with the above INSERT example, but a simpler approach is to SELECT ... FOR UPDATE:

	<?php
	// etc.

	$sql = "SELECT
			   mylob
			FROM
			   mylobs
			WHERE
			   id = 3
			FOR UPDATE /* locks the row */
	";

	$stmt = oci_parse($conn, $sql);

	// Execute the statement using OCI_DEFAULT (begin a transaction)
	oci_execute($stmt, OCI_DEFAULT)
		or die ("Unable to execute query\n");

	// Fetch the SELECTed row
	if ( FALSE === ($row = oci_fetch_assoc($stmt) ) ) {
		oci_rollback($conn);
		die ("Unable to fetch row\n");
	}

	// Discard the existing LOB contents
	if ( !$row['MYLOB']->truncate() ) {
		oci_rollback($conn);
		die ("Failed to truncate LOB\n");
	}

	// Now save a value to the LOB
	if ( !$row['MYLOB']->save('UPDATE: '.date('H:i:s',time()) ) ) {

		// On error, rollback the transaction
		oci_rollback($conn);

	} else {

		// On success, commit the transaction
		oci_commit($conn);

	}

	// Free resources
	oci_free_statement($stmt);
	$row['MYLOB']->free();


	// etc.
	?>

As with the INSERT, I need to perform the UPDATE using a transaction. An important additional step is the call to truncate() . When updating a LOB with save(), it will replace the contents of the LOB beginning from the start up to the length of the new data. That means older content (if it was longer than the new content) may still be left in the LOB.
For PHP 4.x, where truncate() is unavailable, the following alternative solution uses Oracle's EMPTY_CLOB() function to erase any existing contents in the LOB before saving new data to it.

	$sql = "UPDATE
			   mylobs
			SET
				mylob = EMPTY_CLOB()
			WHERE
			   id = 2403
			RETURNING
				mylob INTO :mylob
	";

	$stmt = OCIParse($conn, $sql);

	$mylob = OCINewDescriptor($conn,OCI_D_LOB);

	OCIBindByName($stmt,':mylob',$mylob, -1, OCI_B_CLOB);

	// Execute the statement using OCI_DEFAULT (begin a transaction)
	OCIExecute($stmt, OCI_DEFAULT)
		or die ("Unable to execute query\n");

	if ( !$mylob->save( 'UPDATE: '.date('H:i:s',time()) ) ) {

		OCIRollback($conn);
		die("Unable to update lob\n");

	}

	OCICommit($conn);
	$mylob->free();
	OCIFreeStatement($stmt);

### Working with BFILES
When using the BFILE type, INSERTs and UPDATEs mean telling Oracle where the file is located within the filesystem of the database server (which may not be the same machine as the Web server), rather than passing the file content. Using a SELECT statement, you can read the contents of the BFILE through Oracle, should you so desire, or call functions and procedures from the DBMS_LOB package to get information about the file.

The main advantage of BFILEs is being able to access the original files directly from the filesystem while still being able to locate files using SQL. This means, for example, images can be served directly by the Web server while I can keep track of the relationship between the table containing the BFILES and, say, a "users" table, telling me who uploaded the files.

As an example, I first need to update the table schema used above;

ALTER TABLE mylobs ADD( mybfile BFILE )
Next I need to register a directory alias with Oracle (this requires administrative privileges) and grant permissions to read it:
CREATE DIRECTORY IMAGES_DIR AS '/home/harryf/public_html/images'
GRANT READ ON DIRECTORY IMAGES_DIR TO scott
I can now INSERT some BFILE names like:

	<?php
	// etc.

	// Build an INSERT for the BFILE names
	$sql = "INSERT INTO
			mylobs
			  (
				id,
				mybfile
			  )
		   VALUES
			  (
				mylobs_id_seq.NEXTVAL,
				/*
				Pass the file name using the Oracle directory reference
				I created called IMAGES_DIR
				*/
				BFILENAME('IMAGES_DIR',:filename)
			  )";

	$stmt = oci_parse($conn, $sql);

	// Open the directory
	$dir = '/home/harryf/public_html/images';
	$dh = opendir($dir)
		or die("Unable to open $dir");

	// Loop through the contents of the directory
	while (false !== ( $entry = readdir($dh) ) ) {

		// Match only files with the extension .jpg, .gif or .png
		if ( is_file($dir.'/'.$entry) && preg_match('/\.(jpg|gif|png)$/',$entry) ) {

			// Bind the filename of the statement
			oci_bind_by_name($stmt, ":filename", $entry);

			// Execute the statement
			if ( oci_execute($stmt) ) {
				print "$entry added\n";
			}
		}

	}

If I need to, I can read the BFILE content through Oracle using the same approach as above where I selected CLOBs. Alternatively, if I need to get the filenames back so I can access them directly from the filesystem, I can call the DBMS_LOB.FILEGETNAME procedure like:
	<?php
	// etc.

	$sql = "SELECT
			  id
			FROM
			  mylobs
			WHERE
			  -- Select only BFILES which are not null
			  mybfile IS NOT NULL;

	$stmt1 = oci_parse($conn, $sql);

	oci_execute($stmt1)
		or die ("Unable to execute query\n");

	$sql = "DECLARE
			  locator BFILE;
			  diralias VARCHAR2(30);
			  filename VARCHAR2(30);

			BEGIN

			  SELECT
				mybfile INTO locator
			  FROM
				mylobs
			  WHERE
				id = :id;

			  -- Get the filename from the BFILE
			  DBMS_LOB.FILEGETNAME(locator, diralias, filename);

			  -- Assign OUT params to bind parameters
			  :diralias:=diralias;
			  :filename:=filename;

		   END;";

	$stmt2 = oci_parse($conn, $sql);

	while ( $row = oci_fetch_assoc ($stmt1) ) {

		oci_bind_by_name($stmt2, ":id", $row['ID']);
		oci_bind_by_name ($stmt2, ":diralias", $diralias,30);
		oci_bind_by_name ($stmt2, ":filename", $filename,30);

		oci_execute($stmt2);
		print "{$row['ID']}: $diralias/$filename\n";

	}
	// etc.
	?>

Furthermore, you can use the DBMS_LOB.FILEEXISTS function to discover which files have been deleted via the operating system but are still referenced in the database.
Conclusion

In this HowTo you have been introduced to the different types of LOBs available in Oracle Database 10g and hopefully now understand their role in allowing large data entities to be stored efficiently in the database. You have also learned how to work with LOBs using PHP's OCI8 API, covering the common use cases you will encounter while developing with Oracle and PHP.


