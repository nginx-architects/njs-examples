Authorizing connections using ngx.fetch() as auth_request [stream/auth_request]
===========================================
The example illustrates the usage of ngx.fetch() as an `auth request`_ analog in
stream with a very simple TCP-based protocol: a connection starts with a
magic prefix "MAGiK" followed by a secret 2 bytes. The preread_verify handler
reads the first part of a connection and sends the secret bytes for verification
to a HTTP endpoint. Later it decides based upon the endpoint reply whether
forward the connection to an upstream or reject the connection.


**Step 1:** Use the following commands to start your NGINX container with this lab's files:

.. code-block:: shell

  EXAMPLE='stream/auth_request'
  docker run --rm --name njs_example  -v $(pwd)/conf/$EXAMPLE.conf:/etc/nginx/nginx.conf:ro -v $(pwd)/njs/:/etc/nginx/njs/:ro -p 80:80 -p 443:443 -d nginx

**Step 2:** Now let's use curl to test our NGINX server:

.. code-block:: shell

  telnet 127.0.0.1 80
  ...
  Hi
  Connection closed by foreign host.

  telnet 127.0.0.1 80
  ...
  MAGiKQZ
  BACKEND
  Connection closed by foreign host.

  telnet 127.0.0.1 80
  ...
  MAGiKQQ
  Connection closed by foreign host.

  docker stop njs_example

**Code Snippets**

Notice how this config uses location blocks to define the target of each subrequest.

.. code-block:: nginx

  stream {
        js_path "/etc/nginx/njs/";

        js_import main from stream/auth_request.js;

        server {
              listen 80;

              js_preread  main.preread_verify;

              proxy_pass 127.0.0.1:8081;
        }

        server {
              listen 8081;

              return BACKEND\n;
        }
  }

  http {
        js_path "/etc/nginx/njs/";

        js_import main from stream/auth_request.js;

        server {
              listen 8080;

              server_name  aaa;

              location /validate {
                  js_content main.validate;
              }
        }
  }


This njs code retrieves a token from the "/auth" location and then passes the token to a second subrequest of the "/backend" location.

.. code-block:: js

  function preread_verify(s) {
      var collect = '';

      s.on('upload', function (data, flags) {
          collect += data;

          if (collect.length >= 5 && collect.startsWith('MAGiK')) {
              s.off('upload');
              ngx.fetch('http://127.0.0.1:8080/validate',
                        {body: collect.slice(5,7), headers: {Host:'aaa'}})
              .then(reply => (reply.status == 200) ? s.done(): s.deny())

          } else if (collect.length) {
              s.deny();
          }
      });
  }

  function validate(r) {
          r.return((r.requestText == 'QZ') ? 200 : 403);
  }

  export default {validate, preread_verify};

