.. _PythonAPIs:

******
Python
******

System loggings
===============

.. code-block:: discos

   from IRAPy import logger
	logger.logNotice("My system log message")   
   
Custom loggings
===============

.. code-block:: discos

	from IRAPy import userLogger
	logger.logNotice("My user log message")
   
System exceptions
=================

.. code-block:: discos

	from IRAPy import logger
	ex=MyErrroExImpl()
	logger.logException(ex)

Custom exceptions
=================

.. code-block:: discos

	from IRAPy import userLogger
	ex=MyErrroExImpl()
	userLogger.logException(ex)
   
Interaction with operator input
===============================

.. highlights::

	This function is for components development. The user message is encapsulated with 
	the exception and presented to the user if the error is directly generated by a user command.

.. code-block:: discos

	from IRAPy import logger
	from SimpleParserPy import add_user_message
	ex=MyErrroExImpl()
	add_user_message(ex,"My special message to the user",False)
	raise ex

.. highlights::

	This macro is for stand alone programs that are issued as system calls by the command parser. The user message is 
	handled like the other case and it's also written into standard err and intercepted by the parser to be shown
	to the user.


.. code-block:: discos

	from IRAPy import logger
	from SimpleParserPy import add_user_message	
	newEx = ClientErrorsImpl.CouldntPerformActionExImpl()
	add_user_message(newEx,"My special message to the user")
	userLogger.logException(newEx)
	sys.exit(1)
	
