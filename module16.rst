Converting response body characters to lower case [http/response/to_lower_case]
===============================================================================

**Step 1:** Use the following commands to start your NGINX container with this lab's files:

.. code-block:: shell

  EXAMPLE='http/response/to_lower_case'
  docker run --rm --name njs_example  -v $(pwd)/conf/$EXAMPLE.conf:/etc/nginx/nginx.conf:ro -v $(pwd)/njs/:/etc/nginx/njs/:ro -p 80:80 -p 443:443 -d nginx

**Step 2:** Now let's use curl to test our NGINX server:

.. code-block:: shell

  curl http://localhost/
  hello world

  docker stop njs_example

Code Snippets
~~~~~~~~~~~~~

Notice how this config uses location blocks to define the target of each subrequest.

.. code-block:: nginx

  ...

  http {
    js_path "/etc/nginx/njs/";

    js_import main from http/response/to_lower_case.js;

    server {
          listen 80;

          location / {
              js_body_filter main.to_lower_case;
              proxy_pass http://localhost:8080;
          }
    }

    server {
          listen 8080;

          location / {
              return 200 'Hello World';
          }
    }
  }



This njs code retrieves a token from the "/auth" location and then passes the token to a second subrequest of the "/backend" location.

.. code-block:: js

    function to_lower_case(r, data, flags) {
        r.sendBuffer(data.toLowerCase(), flags);
    }

    export default {to_lower_case};


