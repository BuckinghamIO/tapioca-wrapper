============
New Flavours
============


Wrapping an API with Tapioca
============================

The easiest way to wrap an API using tapioca is starting from the `cookiecutter template <https://github.com/vintasoftware/cookiecutter-tapioca>`_. 

To use it, install cookiecutter in your machine:

.. code-block:: bash

	pip install cookiecutter

and then use it to download the template and run the config steps:

.. code-block:: bash

	cookiecutter gh:vintasoftware/cookiecutter-tapioca

After this process, it's possible that you have a ready to go wrapper. But in most cases you will need to customize stuff. Read through this document to understand what methods are available and how your wrapper can make the most of tapioca. Also, you might want to take a look in the source code of :doc:`other wrappers <flavours>` to get more ideas. 

In case you are having any difficulties, seek help on `Gitter <https://gitter.im/vintasoftware/tapioca-wrapper>`_ or send an email to contact@vinta.com.br .

Adapter
=======

Tapioca features are mainly implemented in the ``TapiocaClient`` and ``TapiocaClientExecutor`` classes. Those are generic classes common to all wrappers and cannot be customized to specific services. All the code specific to the API wrapper you are creating goes in your adapter class, which should inherit from ``TapiocaAdapter`` and implement specific behaviours to the service you are working with. 

Take a look in the generated code from the cookiecutter or in the `tapioca-facebook adapter <https://github.com/vintasoftware/tapioca-facebook/blob/master/tapioca_facebook/tapioca_facebook.py>`_ to get a little familiar with it before you contiue. Note that at the end of the module you will need to perform the transformation of your adapter into a client:

.. code-block:: python

	Facebook = generate_wrapper_from_adapter(FacebookClientAdapter)


Resource Mapping
================

The resource mapping is a very simple dictionary which will tell your tapioca client about the available endpoints and how to call them. There's an example in your cookiecutter generated project. You can also take a look at `tapioca-facebook's resouce mapping <https://github.com/vintasoftware/tapioca-facebook/blob/master/tapioca_facebook/resource_mapping.py>`_.

Tapioca uses `requests <http://docs.python-requests.org/en/latest/>`_ to perform HTTP requests. This is important to know because you will be using the method ``get_request_kwargs`` to set authentication details and return a dictionary that will be passed directly to ther request method. 

Data formatting Mixins
======================

You might want to use one of the following mixins to help you with data format handling in your wrapper: 

- ``FormAdapterMixin`` 
- ``JSONAdapterMixin``


.. class:: TapiocaAdapter

Attributes
----------

.. attribute:: api_root

This should contain the base URL that will be concatenated with the resource mapping itens and generate the final request URL. You can either set this attribute or use the ``get_api_root`` method.

.. attribute:: serializer_class

For more information about the ``serializer_class`` attribute, read the :doc:`serializers documentation <serializers>`.

Methods
-------

.. method:: get_api_root(self, api_params)

This method can be used instead of the ``api_root`` attribute. You might also use it to decide which base URL to use according to a user input.

.. code-block:: python

	def get_api_root(self, api_params):
		if api_params.get('development'):
			return 'http://api.the-dev-url.com/'
		return 'http://api.the-production-url.com/'

.. method:: get_request_kwargs(self, api_params, *args, **kwargs)

This method is called just before any request is made. You should use it to set whatever credentials the request might need. The **api_params** argument is a dictionary and has the parameters passed during the initialization of the tapioca client:

.. code-block:: python
	
	cli = Facebook(access_token='blablabla', client_id='thisistheis')

For this example, api_params will be a dictionary with the keys ``access_token`` and ``client_id``.

Here is an example of how to implement Basic Auth:

.. code-block:: python

	from requests.auth import HTTPBasicAuth

	class MyServiceClientAdapter(TapiocaAdapter):
		...
		def get_request_kwargs(self, api_params, *args, **kwargs):
			params = super(MyServiceClientAdapter, self).get_request_kwargs(
				api_params, *args, **kwargs)

			params['auth'] = HTTPBasicAuth(
				api_params.get('user'), api_params.get('password'))

			return params

.. method:: process_response(self, response)

This method is responsible for converting data returned in a response to a dictionary (which should be returned). It should also be used to raise exceptions when an error message or error response status is returned. [TODO: document exceptions and reference here]

.. method:: format_data_to_request(self, data)

This converts data passed to the body of the request into text. For example, if you need to send JSON, you should use ``json.dumps(data)`` and return the response. **See the mixins section above.**

.. method:: response_to_native(self, response)

This method receives the response of a request and should return a dictionay with the data contained in the response. **see the mixins section above.**

.. method:: get_iterator_next_request_kwargs(self, iterator_request_kwargs, response_data, response)

Override this method if the service you are using supports pagination. It should return a dictionary that will be used to fetch the next batch of data, e.g.:

.. code-block:: python
	
	def get_iterator_next_request_kwargs(self,
			iterator_request_kwargs, response_data, response):
		paging = response_data.get('paging')
		if not paging:
			return
		url = paging.get('next')

		if url:
			iterator_request_kwargs['url'] = url
			return iterator_request_kwargs

In this example, we are updating the URL from the last call made. ``iterator_request_kwargs`` contains the paramenters from the last call made, ``response_data`` contains the response data after it was parsed by ``process_response`` method, and ``response`` is the full response object with all its attributes like headers and status code. 

.. method:: get_iterator_list(self, response_data)

Many APIs enclose the returned list of objects in one of the returned attributes. Use this method to extract and return only the list from the response.

.. code-block:: python

	def get_iterator_list(self, response_data):
		return response_data['data']

In this example, the object list is enclosed in the ``data`` attribute.
