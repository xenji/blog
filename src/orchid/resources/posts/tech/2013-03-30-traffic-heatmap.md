---
title: Visualizing trivago's search traffic on a heatmap

---

While evaluating a message queue for trivago’s architecture I wrote a litte demo to present RabbitMQ in one of our internal meetings. The demo should display live searches from the trivago platform and display them on a map using a heatmap layer. The demo should show that the MQ is able to cope the amount of traffic that would hit the queues and this type of technology solves some of our current near realtime problems.

As I was not allowed to touch the live systems code for it, the data needed to come from somewhere else. The accesslog seemed a proper source of information as our scribe logger delivers it in near-realtime. We use two different parameters to determine the location the visitor wants to search for, one is an integer, the other one is a list of integers. A “nice to have” feature is to show the visitors location. The scripts show this feature already, but the screen recording just shows the destinations. Log parsing? AWK to the rescue!

{% highlight language="awk" %}
function isNoHomepageSearch(input) {
  return input !~ /bIsWorldwide/
}

function getIp(input) {
  return $3
}

function isGetRequest(input) {
  return input ~ /^"GET/
}

function getPathId(input) {
  match(input, /iPathId=([0-9])+/, pathId)

  if (length(pathId) > 0) {
    return pathId[0]
  }
}
function getPathList(input) {
  match(input, /aPathList=([0-9,])+/, pathList)

  if (length(pathList) > 0) {
    return pathList[0]
  }
}

function join(array, start, end, sep, result, i) {
  if (sep == "")
    sep = " "
  else if (sep == SUBSEP) # magic value
    sep = ""

  result = array[start]
  for (i = start + 1; i <= end; i++)
    result = result sep array[i]
  return result
}

BEGIN {
  pathListPart[0] = ""
  pathPart[0] = ""
  use_pathlist = 0
}
{
  if (isNoHomepageSearch($7) && isGetRequest($6)) {

    rawPathId = getPathId($7)

    if (use_pathlist == 1)
      rawPathList = getPathList($7)

    if (rawPathId != "") {
      match(rawPathId, /([0-9]+)/, pathPart)
    }

    if (use_pathlist == 1 && rawPathList != "") {
      match(rawPathList, /([0-9,]+)/, pathListPart)
    }

    if (length(pathPart) > 0 && (use_pathlist == 1 && length(pathListPart) > 0))
      final = pathPart[0] "," pathListPart[0]
    else if (use_pathlist == 1 && length(pathListPart) > 0)
      final = pathListPart[0]
    else
      final = pathPart[0]

    print $3 ";" final
  }
}
{% endhighlight %}

To resolve the IDs to a geo coordinate we did some pre-caching using Redis and a simple `ID => geo` construct.

The AWK script writes it’s result to the STDOUT where a simple node.js scripts gets the content, resolves the geo data from Redis and passes the result to the RabbitMQ exchange. The script already shows the capability to use the geo-lite module and resolve the senders location via geoip lookup.

{% highlight language="javascript" %}
var redis = require("redis"),
    client = redis.createClient();
var amqp = require('amqp');
var amqpc = amqp.createConnection({ host: 'somehost', login: 'guest', password: 'guest', vhost: '/' });
var geoip = require('geoip-lite');

var ll = require("lazylines");
amqpc.on('ready', function () {
    amqpc.exchange('heatmap', {passive: true}, function(e){
        process.stdin.resume();
        var inp = new ll.LineReadStream(process.stdin, 'ascii');
        inp.on("line", function (line) {
            var tmp = ll.chomp(line).split(';'), ip = tmp[0], paths = tmp[1].split(',');
            var geo = geoip.lookup(ip);
            if (geo && geo['ll'])
                for (var i in paths) {
                    client.get(paths[i], function(err, reply) {
                        e.publish('latlong.path', JSON.stringify({
                            src: geo['ll'],
                            dest: JSON.parse(reply)
                        }));
                    });
                }

        });
    });
});
{% endhighlight %}

After sending it to the exchange, the messages are collected from the queue by a simple node.js server daemon. This daemon does only a simple job and pushes the messages from the RabbitMQ via broadcast to all connected websocket clients.

{% highlight language="javascript" %}
var express = require('express');
var app = express();
var server = require('http').createServer(app);
var io = require('socket.io').listen(server);
var amqp = require('amqp');
var amqpc = amqp.createConnection({ host: 'somehost', login: 'guest', password: 'guest', vhost: '/' });

process.on('exit', function() {
    amqpc.close();
});

io.set('log level', 1);
app.use(express.static(__dirname + '/public'));
app.set('views', __dirname + '/views');
app.set('view engine', 'twig');
app.set('twig options', {strict_variables: false});

app.get('/', function (req, res){
    res.render('index', {});
});

amqpc.on('ready', function () {
    amqpc.queue('heatmap', {passive: true}, function(q){
        q.subscribe(function (message) {
            if (message && message.data)
                io.sockets.emit('heatmap', JSON.parse(message.data.toString()));
        });
    });
});
server.listen(3000);
{% endhighlight %}

The frontend is as simple as the rest of the stack. On the client-side we use a bit of jQuery, the wonderful gmap3 library and socket.io.

{% highlight language="javascript" %}
$(document).ready(function () {

    var socket = io.connect('http://' + location.host),
        $body = $('body'),
        body_width = $body.width(),
        body_height = $body.height(),
        count_datapoints = 0,
        $map = $('#map'),
        datapoints = 1500,
        heatmap = new google.maps.visualization.HeatmapLayer({
            data: new google.maps.MVCArray(),
            radius: 15
        }),
        srcmap = new google.maps.visualization.HeatmapLayer({
            data: new google.maps.MVCArray(),
            radius: 15,
            gradient: ['rgba(0, 255, 255, 0)',
                'rgba(0, 255, 255, 1)',
                'rgba(0, 191, 255, 1)',
                'rgba(0, 127, 255, 1)',
                'rgba(0, 63, 255, 1)',
                'rgba(0, 0, 255, 1)',
                'rgba(0, 0, 223, 1)',
                'rgba(0, 0, 191, 1)',
                'rgba(0, 0, 159, 1)',
                'rgba(0, 0, 127, 1)',
                'rgba(63, 0, 91, 1)',
                'rgba(127, 0, 63, 1)',
                'rgba(191, 0, 31, 1)',
                'rgba(255, 0, 0, 1)']
        });

    $map.width(body_width + "px").height(body_height + "px").gmap3({
        map: {
            options: {
                mapTypeId: google.maps.MapTypeId.SATELLITE,
                zoom: 3,
                mapTypeControl: true,
                navigationControl: true,
                scrollwheel: true,
                streetViewControl: false
            }
        }
    });

    var map = $map.gmap3('get');
    heatmap.setMap(map);
    srcmap.setMap(map);

    $span = $('#datapoints');
    $maxdp = $('#datapointborder');
    $maxdp.val(datapoints);

    var borderpoints = datapoints;
    $('#set').on('click', function () {
        borderpoints = Number( $('#datapointborder').val() );
        while (count_datapoints >= borderpoints) {
            heatmap.data.removeAt(0);
      srcmap.data.removeAt(0);
            count_datapoints--;
        }
        $span.text(count_datapoints);
    });

    socket.on('heatmap', function (msg) {
        var src = msg['src'], dest = msg['dest'];
        if (dest == null || dest[0] == null || dest[1] == null) return;
        if (count_datapoints > borderpoints) {
            srcmap.data.removeAt(0);
            heatmap.data.removeAt(0);
        }
        else {
            count_datapoints++;
            $span.text(count_datapoints);
        }
        heatmap.data.push(new google.maps.LatLng(dest[0], dest[1]));
        srcmap.data.push(new google.maps.LatLng(src[0], src[1]));
    });
});
{% endhighlight %}

The only limitation with this small toy is the limitation that the browser gives us. I’ve tried to raise the number of simultaneous data points displayed at the same time, but my MacBook Air (late 2012) went hot on more than 5000 points at the same time. This is related to the interaction with the event-list that controls the heatmap layer.

The average message rate in the video was between 300 and 600 messages per second but my screen recorder is just capable of 10fps, sorry ;).

<iframe allowfullscreen="" frameborder="0" height="303" mozallowfullscreen="" src="https://player.vimeo.com/video/63003863" webkitallowfullscreen="" width="500"></iframe>  

[trivago search destinations on a heatmap](https://vimeo.com/63003863) from [Mario Mueller](https://vimeo.com/user17391252) on [Vimeo](https://vimeo.com/).