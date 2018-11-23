# pubg
PlayerUnknown's Battlegrounds (PUBG) is an online multiplayer battle royale game developed and published by PUBG Corporation, a subsidiary of South Korean video game company Bluehole.
The game is based on previous mods that were created by Brendan "PlayerUnknown" Greene for other games using the film Battle Royale for inspiration, and expanded into a standalone game under Greene's creative direction.
In the game, up to one hundred players parachute onto an island and scavenge for weapons and equipment to kill others while avoiding getting killed themselves.
The available safe area of the game's map decreases in size over time, directing surviving players into tighter areas to force encounters. The last player or team standing wins the round. 
const EventEmitter = require('events');
const  WebSocket  =  require('ws');
const  util = require('util');
const  _ = require("underscore");
const  e = new EventEmitter;   //no extends because it's works pretty strange with es6 classes

var express = require('express');
var bodyParser = require('body-parser')
var app = express();
app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());
app.use(express.static('public'));


const settings = require("./settings.js")

class service {  
    constructor(settings) {
        this.settings = settings;
        let server = 'entry.playbattlegrounds.com:81'; //Use /health to check server health  
        let query = { 'provider': 'steam', 'ticket': settings.ticket, 'playerNetId': settings.steamId, 'cc': 'RU', 'clientGameVersion': settings.v };    
        let finalQuery = 'http://' + server + '/userproxy?' + this.encodeQueryData(query);    
        this.ws  =  new  WebSocket(finalQuery);    
        this.counter = 1000000; //Counter to track callbacks from WS server
        this.cbs = {};

        this.ws.on("message", data => {
            try {
                var j = JSON.parse(data);
            } catch (ex) {
                console.log("ERROR DECODING MSG", ex, data);
                var j = [];
            }
            if (j[0] === 0) {
                //Recived initial pkg?
                e.emit("init", j[4], j[5])
                return;
            }
            if (!j[0]) return console.log("Invalidate pkg", data);
            var pkgId = j[0];
            var emitter = j[2];
            var newData = _.clone(j).slice(2)
            if (_.has(this.cbs, pkgId)) {
                this.cbs[pkgId](...newData);
            } else {
                console.log(`Unhandled pkg from ${emitter}`, newData)
            }
            if (this.settings.debug) console.log("INCOME", j);
            e.emit("msg", data);    
        });

        this.ws.on('open', function(data,  flags)  {
            // Need a little time to switch protocols and etc..
            setTimeout(function() {
                e.emit("connected")    
            }, 1000);
        });    
        this.ws.on('error',  function  incoming(data,  flags)  {
            e.emit("error", data);    
        });     
        this.ws.on('close',  function  incoming(code, data)  {
            e.emit("error", data);    
        });      //Send ping every 30 sec
            
        setInterval(() => {
            this.sendMessage("Ping");    
        }, 30000); //30 sec. taken from lobby script

        setInterval(() => {
            this.cbs = {};
        }, 60000);  //remove all old callbacks to clean memory
    }  
    sendMessage() {
        let arg = [...arguments];
        let newAr = [];
        let cb;
        _.each(arg, ar => {
            if (typeof ar !== "function") newAr.push(ar);
            else cb = ar;
        });
        this.counter++;
        let ourId = _.clone(this.counter);
        if (typeof cb == "function") this.cbs[ourId * -1] = cb;        
        var args = [ourId, null, "UserProxyApi", ...newAr];
        this.ws.send(JSON.stringify(args));
        if (this.settings.debug) console.log("Sending", args);      
    }    
    encodeQueryData(data) {    
        let ret = [];    
        for (let d in data) ret.push(encodeURIComponent(d) + '=' + encodeURIComponent(data[d]));    
        return ret.join('&');  
    }  
};

util.inherits(service, EventEmitter);

//All user stats from all servers with all modes
app.get("/getAllUserStats/:id", function(req, res) {
    var id = req.params.id;
    s.sendMessage("GetUserAllRecord", id, function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result;
        res.send({ success: true, userData });
    });
});

app.post('/getAccountIdByNickname', function(req, res) {
    var steamId = req.body.nickname;
    s.sendMessage("GetBroUserStatesByNickname", [steamId], function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result[0];
        try {
            var accId = userData.AccountId;
            res.send({ success: true, userData });
        } catch (ex) {
            res.send({ success: false, error: 2 });
        }
    });
});

app.post('/getAccountByAccountId', function(req, res) {
    var steamId = req.body.id;
    s.sendMessage("GetBroUserStatesByAccountId", [steamId], function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result[0];
        try {
            var accId = userData.AccountId;
            res.send({ success: true, userData });
        } catch (ex) {
            res.send({ success: false, error: 2 });
        }
    });
});


app.get('/getAccountIdByNickname/:nickname', function(req, res) {
    var steamId = req.params.nickname;
    s.sendMessage("GetBroUserStatesByNickname", [steamId], function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result[0];
        try {
            var accId = userData.AccountId;
            res.send({ success: true, userData });
        } catch (ex) {
            res.send({ success: false, error: 2 });
        }
    });
});


app.post('/getAccountId', function(req, res) {
    var steamId = req.body.id;
    s.sendMessage("GetBroUserStatesBySteamId", [steamId], function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result[0];
        try {
            var accId = userData.AccountId;
            res.send({ success: true, userData });
        } catch (ex) {
            res.send({ success: false, error: 2 });
        }
    });
});

app.post("/getStats", function(req, res) {
    var accountId = req.body.accountId;
    var mode = req.body.mode || "solo";
    var server = req.body.server || "eu";
    s.sendMessage("GetUserRecord", accountId, server, mode, function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result;
        res.send({ success: true, userData });
    });
});

app.post("/getBoard", function(req, res) {
    var type = req.body.type || "Rating";
    var mode = req.body.mode || "solo";
    var server = req.body.server || "eu";

    //Using fake accountId to get leaderbord + user stats to hide our token
    s.sendMessage("GetBroLeaderboard", server, mode, type, "account.59e4ce452ac94e27b02a37ac7a301135", function(isSuccess, result) {
        if (!isSuccess) return res.send({ success: false, error: 1, data: result })
        var userData = result;
        res.send({ success: true, userData });
    });
});

var myAccountId;
var s = new service(settings);
e.on("connected", () => {
    app.listen(settings.port, function() {
        console.log(`App listening on port ${settings.port}!`);
    });
    s.sendMessage("Ping");
});

e.on("error", (data) => {
    console.log("Error connecting:", data)
})

e.on("init", (accountId, accountData) => {
    myAccountId = accountId;
    console.log("Loginned as", accountId)
})

