.. module:: aiohttp.test_utils

.. _aiohttp-testing:

Testing
=======

Testing aiohttp web servers
---------------------------

aiohttp provides plugin for *pytest* making writing web server tests
extremely easy, it also provides :ref:`test framework agnostic
utilities <aiohttp-testing-framework-agnostic-utilities>` for testing
with other frameworks such as :ref:`unittest
<aiohttp-testing-unittest-example>`.

Before starting to write your tests, you may also be interested on
reading :ref:`how to write testable
services<aiohttp-testing-writing-testable-services>` that interact
with the loop.


For using pytest plugin please install pytest-aiohttp_ library:

.. code-block:: shell

   $ pip install pytest-aiohttp

If you don't want to install *pytest-aiohttp* for some reason you may
insert ``pytest_plugins = 'aiohttp.pytest_plugin'`` line into
``conftest.py`` instead for the same functionality.



Provisional Status
~~~~~~~~~~~~~~~~~~

The module is a **provisional**.

*aiohttp* has a year and half period for removing deprecated API
(:ref:`aiohttp-backward-compatibility-policy`).

But for :mod:`aiohttp.test_tools` the deprecation period could be reduced.

Moreover we may break *backward compatibility* without *deprecation
period* for some very strong reason.


The Test Client and Servers
~~~~~~~~~~~~~~~~~~~~~~~~~~~

*aiohttp* test utils provides a scaffolding for testing aiohttp-based
web servers.

They consist of two parts: running test server and making HTTP
requests to this server.

:class:`~aiohttp.test_utils.TestServer` runs :class:`aiohttp.web.Application`
based server, :class:`~aiohttp.test_utils.RawTestServer` starts
:class:`aiohttp.web.Server` low level server.

For performing HTTP requests to these servers you have to create a
test client: :class:`~aiohttp.test_utils.TestClient` instance.

The client incapsulates :class:`aiohttp.ClientSession` by providing
proxy methods to the client for common operations such as
*ws_connect*, *get*, *post*, etc.



Pytest
~~~~~~

.. currentmodule:: pytest_aiohttp

The :data:`aiohttp_client` fixture available from pytest-aiohttp_ plugin
allows you to create a client to make requests to test your app.

A simple would be::

    from aiohttp import web

    async def hello(request):
        return web.Response(text='Hello, world')

    async def test_hello(aiohttp_client):
        app = web.Application()
        app.router.add_get('/', hello)
        client = await aiohttp_client(app)
        resp = await client.get('/')
        assert resp.status == 200
        text = await resp.text()
        assert 'Hello, world' in text


It also provides access to the app instance allowing tests to check the state
of the app. Tests can be made even more succinct with a fixture to create an
app test client::

    import pytest
    from aiohttp import web


    async def previous(request):
        if request.method == 'POST':
            request.app['value'] = (await request.post())['value']
            return web.Response(body=b'thanks for the data')
        return web.Response(
            body='value: {}'.format(request.app['value']).encode('utf-8'))

    @pytest.fixture
    def cli(loop, aiohttp_client):
        app = web.Application()
        app.router.add_get('/', previous)
        app.router.add_post('/', previous)
        return loop.run_until_complete(aiohttp_client(app))

    async def test_set_value(cli):
        resp = await cli.post('/', data={'value': 'foo'})
        assert resp.status == 200
        assert await resp.text() == 'thanks for the data'
        assert cli.server.app['value'] == 'foo'

    async def test_get_value(cli):
        cli.server.app['value'] = 'bar'
        resp = await cli.get('/')
        assert resp.status == 200
        assert await resp.text() == 'value: bar'


Pytest tooling has the following fixtures:

.. data:: aiohttp_server(app, *, port=None, **kwargs)

   A fixture factory that creates
   :class:`~aiohttp.test_utils.TestServer`::

      async def test_f(aiohttp_server):
          app = web.Application()
          # fill route table

          server = await aiohttp_server(app)

   The server will be destroyed on exit from test function.

   *app* is the :class:`aiohttp.web.Application` used
                           to start server.

   *port* optional, port the server is run at, if
   not provided a random unused port is used.

   .. versionadded:: 3.0

   *kwargs* are parameters passed to
                  :meth:`aiohttp.web.AppRunner`

   .. versionchanged:: 3.0
   .. deprecated:: 3.2

      The fixture was renamed from ``test_server`` to ``aiohttp_server``.


.. data:: aiohttp_client(app, server_kwargs=None, **kwargs)
          aiohttp_client(server, **kwargs)
          aiohttp_client(raw_server, **kwargs)

   A fixture factory that creates
   :class:`~aiohttp.test_utils.TestClient` for access to tested server::

      async def test_f(aiohttp_client):
          app = web.Application()
          # fill route table

          client = await aiohttp_client(app)
          resp = await client.get('/')

   *client* and responses are cleaned up after test function finishing.

   The fixture accepts :class:`aiohttp.web.Application`,
   :class:`aiohttp.test_utils.TestServer` or
   :class:`aiohttp.test_utils.RawTestServer` instance.

   *server_kwargs* are parameters passed to the test server if an app
   is passed, else ignored.

   *kwargs* are parameters passed to
   :class:`aiohttp.test_utils.TestClient` constructor.

   .. versionchanged:: 3.0

      The fixture was renamed from ``test_client`` to ``aiohttp_client``.

.. data:: aiohttp_raw_server(handler, *, port=None, **kwargs)

   A fixture factory that creates
   :class:`~aiohttp.test_utils.RawTestServer` instance from given web
   handler.::

      async def test_f(aiohttp_raw_server, aiohttp_client):

          async def handler(request):
              return web.Response(text="OK")

          raw_server = await aiohttp_raw_server(handler)
          client = await aiohttp_client(raw_server)
          resp = await client.get('/')

   *handler* should be a coroutine which accepts a request and returns
   response, e.g.

   *port* optional, port the server is run at, if
   not provided a random unused port is used.

   .. versionadded:: 3.0

.. data:: aiohttp_unused_port()

   Function to return an unused port number for IPv4 TCP protocol::

      async def test_f(aiohttp_client, aiohttp_unused_port):
          port = aiohttp_unused_port()
          app = web.Application()
          # fill route table

          client = await aiohttp_client(app, server_kwargs={'port': port})
          ...

   .. versionchanged:: 3.0

      The fixture was renamed from ``unused_port`` to ``aiohttp_unused_port``.

.. data:: aiohttp_client_cls

   A fixture for passing custom :class:`~aiohttp.test_utils.TestClient` implementations::

      class MyClient(TestClient):
          async def login(self, *, user, pw):
              payload = {"username": user, "password": pw}
              return await self.post("/login", json=payload)

      @pytest.fixture
      def aiohttp_client_cls():
          return MyClient

      def test_login(aiohttp_client):
          app = web.Application()
          client = await aiohttp_client(app)
          await client.login(user="admin", pw="s3cr3t")

   If you want to switch between different clients in tests, you can use
   the usual ``pytest`` machinery. Example with using test markers::

      class RESTfulClient(TestClient):
          ...

      class GraphQLClient(TestClient):
          ...

      @pytest.fixture
      def aiohttp_client_cls(request):
          if request.node.get_closest_marker('rest') is not None:
              return RESTfulClient
          if request.node.get_closest_marker('graphql') is not None:
              return GraphQLClient
          return TestClient


      @pytest.mark.rest
      async def test_rest(aiohttp_client) -> None:
          client: RESTfulClient = await aiohttp_client(Application())
          ...


      @pytest.mark.graphql
      async def test_graphql(aiohttp_client) -> None:
          client: GraphQLClient = await aiohttp_client(Application())
          ...


.. _aiohttp-testing-unittest-example:

.. _aiohttp-testing-unittest-style:

Unittest
~~~~~~~~

.. currentmodule:: aiohttp.test_utils


To test applications with the standard library's unittest or unittest-based
functionality, the AioHTTPTestCase is provided::

    from aiohttp.test_utils import AioHTTPTestCase
    from aiohttp import web

    class MyAppTestCase(AioHTTPTestCase):

        async def get_application(self):
            """
            Override the get_app method to return your application.
            """
            async def hello(request):
                return web.Response(text='Hello, world')

            app = web.Application()
            app.router.add_get('/', hello)
            return app

        async def test_example(self):
            async with self.client.request("GET", "/") as resp:
                self.assertEqual(resp.status, 200)
                text = await resp.text()
            self.assertIn("Hello, world", text)

.. class:: AioHTTPTestCase

    A base class to allow for unittest web applications using aiohttp.

    Derived from :class:`unittest.TestCase`

    Provides the following:

    .. attribute:: client

       an aiohttp test client, :class:`TestClient` instance.

    .. attribute:: server

       an aiohttp test server, :class:`TestServer` instance.

       .. versionadded:: 2.3

    .. attribute:: loop

       The event loop in which the application and server are running.

       .. deprecated:: 3.5

    .. attribute:: app

       The application returned by :meth:`~aiohttp.test_utils.AioHTTPTestCase.get_application`
       (:class:`aiohttp.web.Application` instance).

    .. comethod:: get_client()

       This async method can be overridden to return the :class:`TestClient`
       object used in the test.

       :return: :class:`TestClient` instance.

       .. versionadded:: 2.3

    .. comethod:: get_server()

       This async method can be overridden to return the :class:`TestServer`
       object used in the test.

       :return: :class:`TestServer` instance.

       .. versionadded:: 2.3

    .. comethod:: get_application()

       This async method should be overridden
       to return the :class:`aiohttp.web.Application`
       object to test.

       :return: :class:`aiohttp.web.Application` instance.

    .. comethod:: setUpAsync()

       This async method do nothing by default and can be overridden to execute
       asynchronous code during the ``setUp`` stage of the ``TestCase``.

       .. versionadded:: 2.3

    .. comethod:: tearDownAsync()

       This async method do nothing by default and can be overridden to execute
       asynchronous code during the ``tearDown`` stage of the ``TestCase``.

       .. versionadded:: 2.3

    .. method:: setUp()

       Standard test initialization method.

    .. method:: tearDown()

       Standard test finalization method.


   .. note::

      The ``TestClient``'s methods are asynchronous: you have to
      execute functions on the test client using asynchronous methods.::

         class TestA(AioHTTPTestCase):

             async def test_f(self):
                 async with self.client.get('/') as resp:
                     body = await resp.text()

Patching unittest test cases
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Patching test cases is tricky, when using python older than 3.8  :py:func:`~unittest.mock.patch` does not behave as it has to.
We recommend using :py:mod:`asynctest` that provides :py:func:`~asynctest.patch` that is capable of creating
a magic mock that supports async. It can be used with a decorator as well as with a context manager:

.. code-block:: python
   :emphasize-lines: 1,37,46

    from asynctest.mock import patch as async_patch

    from aiohttp.test_utils import AioHTTPTestCase, unittest_run_loop
    from aiohttp.web_app import Application
    from aiohttp.web_request import Request
    from aiohttp.web_response import Response
    from aiohttp.web_routedef import get


    async def do_something():
        print('something')


    async def ping(request: Request) -> Response:
        await do_something()
        return Response(text='pong')


    class TestApplication(AioHTTPTestCase):
        def get_app(self) -> Application:
            app = Application()
            app.router.add_routes([
                get('/ping/', ping)
            ])

            return app

        @unittest_run_loop
        async def test_ping(self):
            resp = await self.client.get('/ping/')

            self.assertEqual(resp.status, 200)
            self.assertEqual(await resp.text(), 'pong')

        @unittest_run_loop
        async def test_ping_mocked_do_something(self):
            with async_patch('tests.do_something') as do_something_patch:
                resp = await self.client.get('/ping/')

                self.assertEqual(resp.status, 200)
                self.assertEqual(await resp.text(), 'pong')

                self.assertTrue(do_something_patch.called)

        @unittest_run_loop
        @async_patch('tests.do_something')
        async def test_ping_mocked_do_something_decorated(self, do_something_patch):
            resp = await self.client.get('/ping/')

            self.assertEqual(resp.status, 200)
            self.assertEqual(await resp.text(), 'pong')

            self.assertTrue(do_something_patch.called)


Faking request object
---------------------

aiohttp provides test utility for creating fake
:class:`aiohttp.web.Request` objects:
:func:`aiohttp.test_utils.make_mocked_request`, it could be useful in
case of simple unit tests, like handler tests, or simulate error
conditions that hard to reproduce on real server::

    from aiohttp import web
    from aiohttp.test_utils import make_mocked_request

    def handler(request):
        assert request.headers.get('token') == 'x'
        return web.Response(body=b'data')

    def test_handler():
        req = make_mocked_request('GET', '/', headers={'token': 'x'})
        resp = handler(req)
        assert resp.body == b'data'

.. warning::

   We don't recommend to apply
   :func:`~aiohttp.test_utils.make_mocked_request` everywhere for
   testing web-handler's business object -- please use test client and
   real networking via 'localhost' as shown in examples before.

   :func:`~aiohttp.test_utils.make_mocked_request` exists only for
   testing complex cases (e.g. emulating network errors) which
   are extremely hard or even impossible to test by conventional
   way.


.. function:: make_mocked_request(method, path, headers=None, *, \
                                  version=HttpVersion(1, 1), \
                                  closing=False, \
                                  app=None, \
                                  match_info=sentinel, \
                                  reader=sentinel, \
                                  writer=sentinel, \
                                  transport=sentinel, \
                                  payload=sentinel, \
                                  sslcontext=None, \
                                  loop=...)

   Creates mocked web.Request testing purposes.

   Useful in unit tests, when spinning full web server is overkill or
   specific conditions and errors are hard to trigger.

   :param method: str, that represents HTTP method, like; GET, POST.
   :type method: str

   :param path: str, The URL including *PATH INFO* without the host or scheme
   :type path: str

   :param headers: mapping containing the headers. Can be anything accepted
       by the multidict.CIMultiDict constructor.
   :type headers: dict, multidict.CIMultiDict, list of pairs

   :param match_info: mapping containing the info to match with url parameters.
   :type match_info: dict

   :param version: namedtuple with encoded HTTP version
   :type version: aiohttp.protocol.HttpVersion

   :param closing: flag indicates that connection should be closed after
       response.
   :type closing: bool

   :param app: the aiohttp.web application attached for fake request
   :type app: aiohttp.web.Application

   :param writer: object for managing outcoming data
   :type writer: aiohttp.StreamWriter

   :param transport: asyncio transport instance
   :type transport: asyncio.Transport

   :param payload: raw payload reader object
   :type  payload: aiohttp.StreamReader

   :param sslcontext: ssl.SSLContext object, for HTTPS connection
   :type sslcontext: ssl.SSLContext

   :param loop: An event loop instance, mocked loop by default.
   :type loop: :class:`asyncio.AbstractEventLoop`

   :return: :class:`aiohttp.web.Request` object.

   .. versionadded:: 2.3
      *match_info* parameter.

.. _aiohttp-testing-writing-testable-services:

.. _aiohttp-testing-framework-agnostic-utilities:


Framework Agnostic Utilities
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

High level test creation::

    from aiohttp.test_utils import TestClient, TestServer, loop_context
    from aiohttp import request

    # loop_context is provided as a utility. You can use any
    # asyncio.BaseEventLoop class in its place.
    with loop_context() as loop:
        app = _create_example_app()
        with TestClient(TestServer(app), loop=loop) as client:

            async def test_get_route():
                nonlocal client
                resp = await client.get("/")
                assert resp.status == 200
                text = await resp.text()
                assert "Hello, world" in text

            loop.run_until_complete(test_get_route())


If it's preferred to handle the creation / teardown on a more granular
basis, the TestClient object can be used directly::

    from aiohttp.test_utils import TestClient, TestServer

    with loop_context() as loop:
        app = _create_example_app()
        client = TestClient(TestServer(app), loop=loop)
        loop.run_until_complete(client.start_server())
        root = "http://127.0.0.1:{}".format(port)

        async def test_get_route():
            resp = await client.get("/")
            assert resp.status == 200
            text = await resp.text()
            assert "Hello, world" in text

        loop.run_until_complete(test_get_route())
        loop.run_until_complete(client.close())


A full list of the utilities provided can be found at the
:data:`api reference <aiohttp.test_utils>`


Testing API Reference
---------------------

Test server
~~~~~~~~~~~

Runs given :class:`aiohttp.web.Application` instance on random TCP port.

After creation the server is not started yet, use
:meth:`~aiohttp.test_utils.BaseTestServer.start_server` for actual server
starting and :meth:`~aiohttp.test_utils.BaseTestServer.close` for
stopping/cleanup.

Test server usually works in conjunction with
:class:`aiohttp.test_utils.TestClient` which provides handy client methods
for accessing to the server.

.. class:: BaseTestServer(*, scheme='http', host='127.0.0.1', port=None)

   Base class for test servers.

   :param str scheme: HTTP scheme, non-protected ``"http"`` by default.

   :param str host: a host for TCP socket, IPv4 *local host*
      (``'127.0.0.1'``) by default.

   :param int port: optional port for TCP socket, if not provided a
      random unused port is used.

      .. versionadded:: 3.0

   .. attribute:: scheme

      A *scheme* for tested application, ``'http'`` for non-protected
      run and ``'https'`` for TLS encrypted server.

   .. attribute:: host

      *host* used to start a test server.

   .. attribute:: port

      *port* used to start the test server.

   .. attribute:: handler

      :class:`aiohttp.web.Server` used for HTTP requests serving.

   .. attribute:: server

      :class:`asyncio.AbstractServer` used for managing accepted connections.

   .. comethod:: start_server(**kwargs)

      Start a test server.

   .. comethod:: close()

      Stop and finish executed test server.

   .. method:: make_url(path)

      Return an *absolute* :class:`~yarl.URL` for given *path*.


.. class:: RawTestServer(handler, *, scheme="http", host='127.0.0.1')

   Low-level test server (derived from :class:`BaseTestServer`).

   :param handler: a coroutine for handling web requests. The
                   handler should accept
                   :class:`aiohttp.web.BaseRequest` and return a
                   response instance,
                   e.g. :class:`~aiohttp.web.StreamResponse` or
                   :class:`~aiohttp.web.Response`.

                   The handler could raise
                   :class:`~aiohttp.web.HTTPException` as a signal for
                   non-200 HTTP response.

   :param str scheme: HTTP scheme, non-protected ``"http"`` by default.

   :param str host: a host for TCP socket, IPv4 *local host*
      (``'127.0.0.1'``) by default.

   :param int port: optional port for TCP socket, if not provided a
      random unused port is used.

      .. versionadded:: 3.0


.. class:: TestServer(app, *, scheme="http", host='127.0.0.1')

   Test server (derived from :class:`BaseTestServer`) for starting
   :class:`~aiohttp.web.Application`.

   :param app: :class:`aiohttp.web.Application` instance to run.

   :param str scheme: HTTP scheme, non-protected ``"http"`` by default.

   :param str host: a host for TCP socket, IPv4 *local host*
      (``'127.0.0.1'``) by default.

   :param int port: optional port for TCP socket, if not provided a
      random unused port is used.

      .. versionadded:: 3.0

   .. attribute:: app

      :class:`aiohttp.web.Application` instance to run.


Test Client
~~~~~~~~~~~

.. class:: TestClient(app_or_server, *, \
                      scheme='http', host='127.0.0.1', \
                      cookie_jar=None, **kwargs)

   A test client used for making calls to tested server.

   :param app_or_server: :class:`BaseTestServer` instance for making
                         client requests to it.

                         In order to pass an :class:`aiohttp.web.Application`
                         you need to convert it first to :class:`TestServer`
                         first with ``TestServer(app)``.

   :param cookie_jar: an optional :class:`aiohttp.CookieJar` instance,
                      may be useful with
                      ``CookieJar(unsafe=True, treat_as_secure_origin="http://127.0.0.1")``
                      option.

   :param str scheme: HTTP scheme, non-protected ``"http"`` by default.

   :param str host: a host for TCP socket, IPv4 *local host*
      (``'127.0.0.1'``) by default.

   .. attribute:: scheme

      A *scheme* for tested application, ``'http'`` for non-protected
      run and ``'https'`` for TLS encrypted server.

   .. attribute:: host

      *host* used to start a test server.

   .. attribute:: port

      *port* used to start the server

   .. attribute:: server

      :class:`BaseTestServer` test server instance used in conjunction
      with client.

   .. attribute:: app

      An alias for ``self.server.app``. return ``None`` if
      ``self.server`` is not :class:`TestServer`
      instance(e.g. :class:`RawTestServer` instance for test low-level server).

   .. attribute:: session

      An internal :class:`aiohttp.ClientSession`.

      Unlike the methods on the :class:`TestClient`, client session
      requests do not automatically include the host in the url
      queried, and will require an absolute path to the resource.

   .. comethod:: start_server(**kwargs)

      Start a test server.

   .. comethod:: close()

      Stop and finish executed test server.

   .. method:: make_url(path)

      Return an *absolute* :class:`~yarl.URL` for given *path*.

   .. comethod:: request(method, path, *args, **kwargs)

      Routes a request to tested http server.

      The interface is identical to
      :meth:`aiohttp.ClientSession.request`, except the loop kwarg is
      overridden by the instance used by the test server.

   .. comethod:: get(path, *args, **kwargs)

      Perform an HTTP GET request.

   .. comethod:: post(path, *args, **kwargs)

      Perform an HTTP POST request.

   .. comethod:: options(path, *args, **kwargs)

      Perform an HTTP OPTIONS request.

   .. comethod:: head(path, *args, **kwargs)

      Perform an HTTP HEAD request.

   .. comethod:: put(path, *args, **kwargs)

      Perform an HTTP PUT request.

   .. comethod:: patch(path, *args, **kwargs)

      Perform an HTTP PATCH request.

   .. comethod:: delete(path, *args, **kwargs)

      Perform an HTTP DELETE request.

   .. comethod:: ws_connect(path, *args, **kwargs)

      Initiate websocket connection.

      The api corresponds to :meth:`aiohttp.ClientSession.ws_connect`.


Utilities
~~~~~~~~~

.. function:: make_mocked_coro(return_value)

  Creates a coroutine mock.

  Behaves like a coroutine which returns *return_value*.  But it is
  also a mock object, you might test it as usual
  :class:`~unittest.mock.Mock`::

      mocked = make_mocked_coro(1)
      assert 1 == await mocked(1, 2)
      mocked.assert_called_with(1, 2)


  :param return_value: A value that the the mock object will return when
      called.
  :returns: A mock object that behaves as a coroutine which returns
      *return_value* when called.


.. function:: unused_port()

   Return an unused port number for IPv4 TCP protocol.

   :return int: ephemeral port number which could be reused by test server.

.. function:: loop_context(loop_factory=<function asyncio.new_event_loop>)

   A contextmanager that creates an event_loop, for test purposes.

   Handles the creation and cleanup of a test loop.

.. function:: setup_test_loop(loop_factory=<function asyncio.new_event_loop>)

   Create and return an :class:`asyncio.AbstractEventLoop` instance.

   The caller should also call teardown_test_loop, once they are done
   with the loop.

   .. note::

      As side effect the function changes asyncio *default loop* by
      :func:`asyncio.set_event_loop` call.

      Previous default loop is not restored.

      It should not be a problem for test suite: every test expects a
      new test loop instance anyway.

   .. versionchanged:: 3.1

      The function installs a created event loop as *default*.

.. function:: teardown_test_loop(loop)

   Teardown and cleanup an event_loop created by setup_test_loop.

   :param loop: the loop to teardown
   :type loop: asyncio.AbstractEventLoop



.. _pytest: http://pytest.org/latest/
.. _pytest-aiohttp: https://pypi.python.org/pypi/pytest-aiohttp
