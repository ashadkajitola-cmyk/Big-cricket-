/**
 * CRICKET SCHEDULE & WALLET APP - COMPLETE SINGLE FILE CODE
 * Features:
 * - Admin Number: 9569981484 (Free Admin Panel, 10,00,000 Wallet Bonus)
 * - Other Users: 100 Rs Wallet Bonus, require Subscription to create schedules ("Apna Schedule")
 * - Subscriptions: 4 hours (99), 1 Day (108), 1 Match (200), Half Monthly (299), Monthly (449), Yearly (4000)
 * - Ticket buying, Zomato-style district view, Active tickets, Match Betting (Satta) with double reward system
 * - Dynamic Admin Management (Change admin/free number via Edit Panel)
 * - 6-digit Ticket Code Verification, Ticket limits, and Series structuring.
 */

const express = require('express');
const bodyParser = require('body-parser');
const path = require('path');

const app = express();
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());

// In-Memory Database (Production me isko MongoDB/MySQL se replace kar sakte hain)
let db = {
    users: {
        "9569981484": { phone: "9569981484", wallet: 1000000, activeDevice: null, subscription: { type: 'lifetime', expires: null, matchesLeft: 9999 } },
        "4787759595": { phone: "4787759595", wallet: 5000, activeDevice: null, subscription: { type: 'lifetime', expires: null, matchesLeft: 9999 } }
    },
    settings: {
        mainAdmin: "9569981484",
        subRates: {
            "4hour": 99,
            "1day": 108,
            "1match": 200,
            "halfmonthly": 299,
            "monthly": 449,
            "yearly": 4000
        }
    },
    tickets: [], // { id, code, seriesName, team1, team2, venue, gateTime, price, date, time, totalLimit, soldCount, creator, isInternational }
    activeTickets: [], // { buyId, userPhone, ticketId, teamChosen, amountPaid, status }
    matches: [] // { matchId, seriesName, team1, team2, score1, score2, venue, date, time, result, isNewSeries }
};

// --- HTML FRONTEND GENERATOR (Single Page Application) ---
app.get('/', (req, res) => {
    res.send(`
    <!DOCTYPE html>
    <html lang="hi">
    <head>
        <meta charset="UTF-8">
        <title>Cricket Schedule & Wallet App</title>
        <style>
            body { font-family: Arial, sans-serif; background: #f4f4f9; margin: 0; padding: 20px; color: #333; }
            .container { max-width: 900px; margin: auto; background: #fff; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
            h2, h3 { color: #0288d1; }
            .box { background: #e3f2fd; padding: 15px; margin-bottom: 15px; border-radius: 5px; }
            input, select, button { padding: 10px; margin: 5px 0; width: 100%; box-sizing: border-box; }
            button { background: #0288d1; color: white; border: none; cursor: pointer; border-radius: 4px; font-weight: bold; }
            button:hover { background: #0277bd; }
            .ticket-card { background: #fffde7; border: 1px dashed #fbc02d; padding: 10px; margin: 10px 0; border-radius: 5px; }
            .flex { display: flex; gap: 10px; }
        </style>
    </head>
    <body>
        <div class="container">
            <h2>🏏 Cricket Schedule, Ticket & Satta Platform</h2>
            
            <!-- LOGIN SECTION -->
            <div class="box" id="loginBox">
                <h3>Login / Switch Number</h3>
                <input type="text" id="phoneInput" placeholder="Enter Mobile Number (e.g. 9569981484)">
                <button onclick="loginUser()">Login</button>
                <p id="userInfo"></p>
            </div>

            <!-- DASHBOARD SECTION (Hidden by default) -->
            <div id="dashboard" style="display:none;">
                <div class="box">
                    <h3>👤 Welcome, <span id="dispPhone"></span> | Wallet: ₹<span id="dispWallet"></span></h3>
                    <button onclick="logout()">Logout</button>
                </div>

                <!-- ADMIN & CREATOR CONTROLS -->
                <div class="box" id="creatorPanel" style="display:none;">
                    <h3>🛠️ Create Match / Ticket Schedule</h3>
                    <input type="text" id="seriesName" placeholder="Series Name">
                    <input type="text" id="team1" placeholder="Team 1 Name">
                    <input type="text" id="team2" placeholder="Team 2 Name">
                    <input type="text" id="venue" placeholder="Venue">
                    <input type="text" id="gateTime" placeholder="Gate Open Time">
                    <input type="number" id="ticketPrice" placeholder="Ticket Price (₹)">
                    <input type="number" id="ticketLimit" placeholder="Total Tickets Limit">
                    <input type="date" id="matchDate">
                    <input type="time" id="matchTime">
                    <input type="text" id="ticketCode" placeholder="6-Digit Verification Code">
                    <select id="seriesType">
                        <option value="same">Same Series (Next Match)</option>
                        <option value="new">New Series (With Gap)</option>
                    </select>
                    <button onclick="createTicketSchedule()">Publish Ticket & Schedule</button>
                </div>

                <!-- SUBSCRIPTION BUYING FOR USERS -->
                <div class="box" id="subPanel" style="display:none;">
                    <h3>📦 Buy Creator Subscription (Required for Users)</h3>
                    <select id="subType">
                        <option value="4hour">4 Hour Subscription - ₹99</option>
                        <option value="1day">1 Day Subscription - ₹108 (1 Match Only)</option>
                        <option value="1match">1 Match Subscription - ₹200</option>
                        <option value="halfmonthly">Half Monthly Subscription - ₹299</option>
                        <option value="monthly">Monthly Subscription - ₹449</option>
                        <option value="yearly">Yearly Subscription - ₹4000</option>
                    </select>
                    <button onclick="buySubscription()">Buy Subscription</button>
                </div>

                <!-- VIEW SCHEDULES -->
                <div class="box">
                    <h3>📅 Schedules</h3>
                    <div class="flex">
                        <button onclick="loadSchedules('international')">International Schedule (Admin)</button>
                        <button onclick="loadSchedules('personal')">Apna Schedule (User)</button>
                    </div>
                    <div id="scheduleList"></div>
                </div>

                <!-- CODE VERIFIER -->
                <div class="box">
                    <h3>🔍 6-Digit Code Ticket Checker</h3>
                    <input type="text" id="checkCode" placeholder="Enter 6-Digit Code">
                    <button onclick="verifyCode()">Check Details</button>
                    <div id="codeResult"></div>
                </div>

                <!-- ZOMATO STYLE DISTRICT / ACTIVE TICKETS -->
                <div class="box">
                    <h3>🎫 Buy Tickets & Active Satta Board</h3>
                    <div id="marketList"></div>
                    <h4>My Active Tickets & Satta</h4>
                    <div id="activeTicketList"></div>
                </div>

                <!-- EDIT PANEL (ADMIN ONLY) -->
                <div class="box" id="editPanel" style="display:none;">
                    <h3>⚙️ Master Edit Panel (Admin Only)</h3>
                    <input type="text" id="newAdminPhone" placeholder="Set Free Admin Phone Number">
                    <button onclick="updateAdmin()">Update Admin Number</button>
                    <p>Change Subscription Rates:</p>
                    <input type="number" id="rate4hour" placeholder="4 Hour Rate">
                    <input type="number" id="rate1day" placeholder="1 Day Rate">
                    <button onclick="updateRates()">Update Rates</button>
                </div>
            </div>
        </div>

        <script>
            let currentUser = null;

            function loginUser() {
                const phone = document.getElementById('phoneInput').value;
                if(!phone) return alert('Enter phone number!');
                
                fetch('/api/login', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ phone })
                })
                .then(res => res.json())
                .then(data => {
                    if(data.success) {
                        currentUser = data.user;
                        document.getElementById('loginBox').style.display = 'none';
                        document.getElementById('dashboard').style.display = 'block';
                        document.getElementById('dispPhone').innerText = currentUser.phone;
                        document.getElementById('dispWallet').innerText = currentUser.wallet;
                        
                        // Check permissions
                        if(data.isAdmin || currentUser.subscription.expires || currentUser.subscription.matchesLeft > 0) {
                            document.getElementById('creatorPanel').style.display = 'block';
                        } else {
                            document.getElementById('subPanel').style.display = 'block';
                        }

                        if(data.isAdmin) {
                            document.getElementById('editPanel').style.display = 'block';
                        }
                        loadMarket();
                        loadActiveTickets();
                    } else {
                        alert(data.message);
                    }
                });
            }

            function logout() {
                currentUser = null;
                document.getElementById('loginBox').style.display = 'block';
                document.getElementById('dashboard').style.display = 'none';
            }

            function createTicketSchedule() {
                const payload = {
                    phone: currentUser.phone,
                    seriesName: document.getElementById('seriesName').value,
                    team1: document.getElementById('team1').value,
                    team2: document.getElementById('team2').value,
                    venue: document.getElementById('venue').value,
                    gateTime: document.getElementById('gateTime').value,
                    price: document.getElementById('ticketPrice').value,
                    limit: document.getElementById('ticketLimit').value,
                    date: document.getElementById('matchDate').value,
                    time: document.getElementById('matchTime').value,
                    code: document.getElementById('ticketCode').value,
                    seriesType: document.getElementById('seriesType').value
                };

                fetch('/api/create-schedule', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify(payload)
                }).then(res => res.json()).then(data => {
                    alert(data.message);
                    loadMarket();
                });
            }

            function buySubscription() {
                const type = document.getElementById('subType').value;
                fetch('/api/buy-sub', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ phone: currentUser.phone, type })
                }).then(res => res.json()).then(data => {
                    alert(data.message);
                    location.reload();
                });
            }

            function loadMarket() {
                fetch('/api/market').then(res => res.json()).then(tickets => {
                    let html = '';
                    tickets.forEach(t => {
                        html += \`<div class="ticket-card">
                            <strong>\${t.seriesName}</strong>: \${t.team1} vs \${t.team2}<br>
                            Venue: \${t.venue} | Price: ₹\${t.price} | Left: \${t.totalLimit - t.soldCount}<br>
                            <button onclick="buyTicket('\${t.id}')">Buy Ticket</button>
                        </div>\`;
                    });
                    document.getElementById('marketList').innerHTML = html;
                });
            }

            function buyTicket(ticketId) {
                fetch('/api/buy-ticket', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ phone: currentUser.phone, ticketId })
                }).then(res => res.json()).then(data => {
                    alert(data.message);
                    location.reload();
                });
            }

            function loadActiveTickets() {
                fetch('/api/active-tickets?phone=' + currentUser.phone).then(res => res.json()).then(tickets => {
                    let html = '';
                    tickets.forEach(at => {
                        html += \`<div class="ticket-card">
                            Ticket: \${at.seriesName} (\${at.team1} vs \${at.team2})<br>
                            Select Team for Satta: 
                            <select id="sattaTeam_\${at.buyId}">
                                <option value="\${at.team1}">\${at.team1}</option>
                                <option value="\${at.team2}">\${at.team2}</option>
                            </select>
                            <input type="number" id="sattaAmt_\${at.buyId}" placeholder="Enter Satta Amount (₹)">
                            <button onclick="placeSatta('\${at.buyId}')">Confirm Satta</button>
                        </div>\`;
                    });
                    document.getElementById('activeTicketList').innerHTML = html;
                });
            }

            function placeSatta(buyId) {
                const team = document.getElementById('sattaTeam_' + buyId).value;
                const amount = document.getElementById('sattaAmt_' + buyId).value;
                fetch('/api/place-satta', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ phone: currentUser.phone, buyId, team, amount })
                }).then(res => res.json()).then(data => {
                    alert(data.message);
                    location.reload();
                });
            }

            function loadSchedules(type) {
                fetch('/api/schedules?type=' + type).then(res => res.json()).then(data => {
                    let html = '';
                    data.forEach(s => {
                        html += \`<div class="ticket-card"><b>\${s.seriesName}</b>: \${s.team1} vs \${s.team2} @ \${s.venue}</div>\`;
                    });
                    document.getElementById('scheduleList').innerHTML = html;
                });
            }

            function verifyCode() {
                const code = document.getElementById('checkCode').value;
                fetch('/api/verify-code?code=' + code).then(res => res.json()).then(data => {
                    if(data.success) {
                        document.getElementById('codeResult').innerHTML = \`Match: \${data.ticket.team1} vs \${data.ticket.team2}, Series: \${data.ticket.seriesName}, Venue: \${data.ticket.venue}\`;
                    } else {
                        document.getElementById('codeResult').innerHTML = 'Invalid Code!';
                    }
                });
            }

            function updateAdmin() {
                const newAdmin = document.getElementById('newAdminPhone').value;
                fetch('/api/update-admin', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({ phone: currentUser.phone, newAdmin })
                }).then(res => res.json()).then(data => alert(data.message));
            }
        </script>
    </body>
    </html>
    `);
});

// --- BACKEND API ENDPOINTS ---

app.post('/api/login', (req, res) => {
    const { phone } = req.body;
    if (!db.users[phone]) {
        let bonus = (phone === db.settings.mainAdmin) ? 1000000 : 100;
        db.users[phone] = {
            phone,
            wallet: bonus,
            activeDevice: phone,
            subscription: phone === db.settings.mainAdmin ? { type: 'lifetime', matchesLeft: 9999 } : { type: 'none', matchesLeft: 0 }
        };
    }
    const isAdmin = (phone === db.settings.mainAdmin);
    res.json({ success: true, user: db.users[phone], isAdmin });
});

app.post('/api/create-schedule', (req, res) => {
    const { phone, seriesName, team1, team2, venue, gateTime, price, limit, date, time, code, seriesType } = req.body;
    const isInternational = (phone === db.settings.mainAdmin);

    const newTicket = {
        id: 'TKT_' + Date.now(),
        code,
        seriesName,
        team1,
        team2,
        venue,
        gateTime,
        price: parseFloat(price),
        totalLimit: parseInt(limit),
        soldCount: 0,
        date,
        time,
        creator: phone,
        isInternational
    };

    db.tickets.push(newTicket);
    db.matches.push({
        matchId: 'M_' + Date.now(),
        seriesName,
        team1,
        team2,
        score1: '0/0',
        score2: '0/0',
        venue,
        date,
        time,
        result: null,
        isNewSeries: seriesType === 'new',
        creator: phone
    });

    res.json({ success: true, message: 'Ticket & Schedule created successfully!' });
});

app.post('/api/buy-sub', (req, res) => {
    const { phone, type } = req.body;
    const cost = db.settings.subRates[type] || 100;
    
    if (db.users[phone].wallet < cost) {
        return res.json({ success: false, message: 'Insufficient balance in wallet!' });
    }

    db.users[phone].wallet -= cost;
    // Route money to main admin/creator wallet (9569981484)
    if (db.users[db.settings.mainAdmin]) {
        db.users[db.settings.mainAdmin].wallet += cost;
    }

    if (type === '1match') {
        db.users[phone].subscription = { type, matchesLeft: 1 };
    } else {
        db.users[phone].subscription = { type, matchesLeft: 999 };
    }

    res.json({ success: true, message: 'Subscription purchased successfully!' });
});

app.get('/api/market', (req, res) => {
    res.json(db.tickets);
});

app.post('/api/buy-ticket', (req, res) => {
    const { phone, ticketId } = req.body;
    const ticket = db.tickets.find(t => t.id === ticketId);
    
    if (!ticket || ticket.soldCount >= ticket.totalLimit) {
        return res.json({ success: false, message: 'Ticket not available or sold out!' });
    }

    if (db.users[phone].wallet < ticket.price) {
        return res.json({ success: false, message: 'Not enough money in wallet!' });
    }

    db.users[phone].wallet -= ticket.price;
    // Ticket price goes to main admin 9569981484
    if (db.users[db.settings.mainAdmin]) {
        db.users[db.settings.mainAdmin].wallet += ticket.price;
    }

    ticket.soldCount++;
    
    db.activeTickets.push({
        buyId: 'BUY_' + Date.now(),
        userPhone: phone,
        ticketId,
        seriesName: ticket.seriesName,
        team1: ticket.team1,
        team2: ticket.team2,
        teamChosen: null,
        amountPaid: 0
    });

    res.json({ success: true, message: 'Ticket purchased and moved to Active Tickets!' });
});

app.get('/api/active-tickets', (req, res) => {
    const { phone } = req.query;
    const userTickets = db.activeTickets.filter(at => at.userPhone === phone && at.amountPaid === 0);
    res.json(userTickets);
});

app.post('/api/place-satta', (req, res) => {
    const { phone, buyId, team, amount } = req.body;
    const amt = parseFloat(amount);

    if (db.users[phone].wallet < amt) {
        return res.json({ success: false, message: 'Insufficient wallet balance for Satta!' });
    }

    db.users[phone].wallet -= amt;
    // Deduct amount and add to main admin wallet
    if(db.users[db.settings.mainAdmin]) {
        db.users[db.settings.mainAdmin].wallet += amt;
    }

    const buyRecord = db.activeTickets.find(at => at.buyId === buyId);
    if(buyRecord) {
        buyRecord.teamChosen = team;
        buyRecord.amountPaid = amt;
    }

    res.json({ success: true, message: 'Satta placed successfully!' });
});

app.get('/api/schedules', (req, res) => {
    const { type } = req.query;
    if (type === 'international') {
        res.json(db.matches.filter(m => m.creator === db.settings.mainAdmin));
    } else {
        res.json(db.matches.filter(m => m.creator !== db.settings.mainAdmin));
    }
});

app.get('/api/verify-code', (req, res) => {
    const { code } = req.query;
    const ticket = db.tickets.find(t => t.code === code);
    if(ticket) {
        res.json({ success: true, ticket });
    } else {
        res.json({ success: false });
    }
});

app.post('/api/update-admin', (req, res) => {
    const { phone, newAdmin } = req.body;
    if (phone === db.settings.mainAdmin) {
        db.settings.mainAdmin = newAdmin;
        if (!db.users[newAdmin]) {
            db.users[newAdmin] = { phone: newAdmin, wallet: 1000000, subscription: { type: 'lifetime', matchesLeft: 9999 } };
        }
        res.json({ success: true, message: 'Admin number updated successfully!' });
    } else {
        res.json({ success: false, message: 'Unauthorized action!' });
    }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Cricket Schedule App running on port ${PORT}`);
});
