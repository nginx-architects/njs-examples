Modifying or deleting cookies sent by the upstream server [http/response/modify_set_cookie]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~




**Step 1:** Use the following commands to start your NGINX container with this lab's files:

.. code-block:: shell

  EXAMPLE='http/response/modify_set_cookie'
  docker run --rm --name njs_example  -v $(pwd)/conf/$EXAMPLE.conf:/etc/nginx/nginx.conf:ro -v $(pwd)/njs/:/etc/nginx/njs/:ro -p 80:80 -p 443:443 -d nginx

**Step 2:** Now let's use curl to test our NGINX server:

.. code-block:: shell

  curl http://localhost/modify_cookies?len=1 -v
    ...
  < Set-Cookie: XXXXXX
  < Set-Cookie: BB
  < Set-Cookie: YYYYYYY

  curl http://localhost/modify_cookies?len=3 -v
    ...
  < Set-Cookie: XXXXXX
  < Set-Cookie: YYYYYYY

  docker stop njs_example

**Code Snippets**

Notice how this config uses location blocks to define the target of each subrequest.

.. code-block:: nginx

  ...

  http {
    js_path "/etc/nginx/njs/";

    js_import main from http/response/modify_set_cookie.js;

    server {
          listen 80;

          location /modify_cookies {
              js_header_filter main.cookies_filter;
              proxy_pass http://localhost:8080;
          }
    }

    server {
          listen 8080;

          location /modify_cookies {
              add_header Set-Cookie "XXXXXX";
              add_header Set-Cookie "BB";
              add_header Set-Cookie "YYYYYYY";
              return 200;
          }
    }
  }



This njs code retrieves a token from the "/auth" location and then passes the token to a second subrequest of the "/backend" location.

.. code-block:: js

    function cookies_filter(r) {
        var cookies = r.headersOut['Set-Cookie'];
        r.headersOut['Set-Cookie'] = cookies.filter(v=>v.length > Number(r.args.len));
    }

    export default {cookies_filter};

