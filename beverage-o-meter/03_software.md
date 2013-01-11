!SLIDE

# What sort of softare is needed? #

!SLIDE bullets

# Softare #

* Raspbian “wheezy”
* python
* sqlite
* tmux
* wicd (WIFI Management)
* lighttpd

!SLIDE

Followed the instructions on the RaspberyPI site to install "wheezy".

Installed everything else via apt-get and python's package manager pip.

!SLIDE

# Code #

We needed some code in order for this thing to work.

!SLIDE

# `bom_barcode_reader.py` #

This runs forever waiting for keyboard input and saving the scans in the
database

    @@@ python
    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
     
    from datetime import datetime
    import sqlite3 as lite
    import sys
     
    con = lite.connect('bom.db')

!SLIDE

    @@@ python
    with con:
      cur = con.cursor()
      cur.execute(
        """
        CREATE TABLE IF NOT EXISTS upc(
            id INTEGER PRIMARY KEY
             AUTOINCREMENT,
            upc TEXT,
            `timestamp` timestamp DEFAULT
             CURRENT_TIMESTAMP
        )
        """
      )
     
    con.commit()
    con.close()

!SLIDE

    @@@ python 
    while True:
      barcode = raw_input(
      	"scann bar code: "
      )
     
      if barcode == '':
        continue
     
      print datetime.now()
      print barcode

!SLIDE

    @@@ python 
      con = lite.connect('bom.db')
      with con:
        cur = con.cursor()
        cur.execute(
            "INSERT INTO upc (upc) VALUES( ? )"
            , (barcode,)
        )
     
      con.commit()
      con.close()

!SLIDE

* The `bom` user was setup with auto login.
* The following went in `dom`'s' `.bash_profile`.

# `.bash_profile` #

    @@@ bash
    tmux new-session -s BOM \
     "python bom_barcodereader.py"$'\n'

!SLIDE

	@@@ python
    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
     
    import web
    import json
    from pprint import pprint
     
    urls = (
      '/', 'index',
      '/after/(.*)', 'after',
    )

!SLIDE

    @@@ python
    db = web.database(
      dbn="sqlite"
      , db="bom.db"
    )
    
    def json_encode_handler(obj):
        if hasattr(obj, 'isoformat'):
            return obj.isoformat()
        else:
            raise TypeError, 'Object of type'
            ' %s with value of %s is not JSON'
            ' serializable' % (
              type(obj)
              , repr(obj)
            ) 
 
!SLIDE

    @@@ python
    class index:
        def GET(self):
     
            upcs = []
     
            for upc in db.select('upc'):
                upcs.append(upc)
     
            web.header(
              'Content-Type'
              , 'application/json'
             )
            return json.dumps(
              upcs
              , default=json_encode_handler
             )

!SLIDE small

    @@@ python
    class after:
        def GET(self, after_id):
            upcs = []
            vars = dict(id = after_id)
            rows = db.select(
              'upc'
              , vars
              , where="id > $id"
            )
            for upc in rows:
                upcs.append(upc)
     
            web.header(
              'Content-Type'
              , 'application/json'
            )
            return json.dumps(
              upcs
              , default=json_encode_handler
            )

!SLIDE

    @@@ python
    if __name__ == "__main__": 
        app = web.application(urls, globals())
        app.run()

!SLIDE small

# Webserver #

# `/etc/lighttpd/conf-enabled/10-fastcgi.conf` #

    server.modules += ( "mod_fastcgi" )
    server.modules   += ( "mod_rewrite" )
    
    fastcgi.server = ( "/bom_server.py" =>
     (( "socket" => "/tmp/fastcgi.socket",
        "bin-path" => "/home/bom/bom_server.py",
        "max-procs" => 1,
       "bin-environment" => (
         "REAL_SCRIPT_NAME" => ""
       ),
       "check-local" => "disable"
     ))
     )

!SLIDE small

     url.rewrite-once = (
       "^/favicon.ico$" => "/static/favicon.ico",
       "^/static/(.*)$" => "/static/$1",
       "^/(.*)$" => "/bom_server.py/$1",
     )
