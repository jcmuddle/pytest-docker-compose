pytest-docker-compose
=====================
This package contains a `pytest`_ plugin for integrating Docker Compose into
your automated integration tests.

Given a path to a ``docker-compose.yml`` file, it will automatically build the
project at the start of the test run, bring the containers up before each test
starts, and tear them down after each test ends.


Dependencies
------------
Make sure you have `Docker`_ installed.

This plugin has been tested with Python 3.6 and pytest 3.4.

.. note:: This plugin is not compatible with Python 2.


Installation
------------
Install the plugin using pip::

    > pip install pytest-docker-compose


Usage
-----
For performance reasons, the plugin is not enabled by default, so you must
activate it manually in the tests that use it:

.. code-block:: python

    pytest_plugins = ["docker_compose"]

To interact with Docker containers in your tests, use the following fixtures:

``docker_containers``
    A list of the Docker ``compose.container.Container`` objects running during
    the test.

``docker_network_info``
    A list of ``pytest_docker_compose.NetworkInfo`` objects for each container,
    grouped by service name.

    ``NetworkInfo`` is a namedtuple with ``hostname`` and ``port`` fields,
    allowing you to build HTTP and other URLs to interact with each container.

.. tip::
    You can integrate these into a project-specific fixture to return a
    configured client object, depending on what services your containers expose.

    For example:

    .. code-block:: python

        import pytest
        import typing
        from pytest_docker_compose import NetworkInfo

        from my_app import ApiClient

        pytest_plugins = ["docker_compose"]

        @pytest.fixture(name="api_client")
        def fixture_api_client(
                docker_network_info: typing.Dict[str, typing.List[NetworkInfo]],
        ) -> ApiClient:
            # ``docker_network_info`` is grouped by service name.
            service = docker_network_info["my_api_service"][0]

            return ApiClient(
                base_url=f"http://{service.hostname}:{service.port}/api/v1",
            )


        # Tests can then interact with the API client directly.
        def test_health_check(api_client: ApiClient):
            assert api_client.health_check() == {"status": "ok"}


Running Integration Tests
-------------------------
Use `pytest`_ to run your tests as normal:

.. code-block:: sh

    pytest

By default, this will look for a ``docker-compose.yml`` file in the current
working directory.  You can specify a different file via the
``--docker-compose`` option:

.. code-block:: sh

    pytest --docker-compose=/path/to/docker-compose.yml

.. tip::
    Alternatively, you can specify this option in your ``pytest.ini`` file:

    .. code-block:: ini

        [pytest]
        addopts = --docker-compose=/path/to/docker-compose.yml

    The option will be ignored for tests that do not use this plugin.

    See `Configuration Options`_ for more information on using configuration
    files to modify pytest behavior.


Waiting for Services to Come Online
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The fixture will wait until every container is up before handing control over to
the test.

However, just because a container is up does not mean that the services running
on it are ready to accept incoming requests yet!

If your tests need to wait for a particular condition (for example, to wait for
an HTTP health check endpoint to send back a 200 response), define a
``docker_startup_check`` fixture in your ``conftest.py`` file.

Here's a simple example of a fixture that waits for an HTTP service to come
online before starting each test.

.. code-block:: python

    import pytest
    import requests
    import typing
    from pytest_docker_compose import NetworkInfo
    from time import sleep, time

    @pytest.fixture
    def docker_startup_check
            docker_network_info: typing.Dict[str, typing.List[NetworkInfo]],
    ) -> typing.NoReturn:
        start = time()
        timeout = 5

        for name, network_info in docker_network_info.items():
            while True:
                if time() - start >= timeout:
                    raise RuntimeError(
                        f"Unable to start all container services "
                        "within {timeout} seconds.",
                    )

                url = f"http://{network_info.hostname}:{network_info.port}/health_check"

                try:
                    if requests.get(url).status_code == 200:
                        break
                except ConnectionError:
                    pass

                sleep(0.1)


.. _Configuration Options: https://docs.pytest.org/en/latest/customize.html#adding-default-options
.. _Docker: https://www.docker.com/
.. _pytest: https://docs.pytest.org/