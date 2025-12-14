import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpExchange;
import java.io.*;
import java.net.InetSocketAddress;
import java.time.LocalDate;
import java.util.*;
import java.util.stream.Collectors;

class Venue {
    public Long id;
    public String name;
    public String location;
    public String description;
    public Double pricePerHour;
    public Integer capacity;

    public Venue(Long id, String name, String location, String description, Double pricePerHour, Integer capacity) {
        this.id = id;
        this.name = name;
        this.location = location;
        this.description = description;
        this.pricePerHour = pricePerHour;
        this.capacity = capacity;
    }
}

class Booking {
    public Long id;
    public Long venueId;
    public String venueName;
    public String userName;
    public String bookingDate;
    public Integer hours;
    public Double totalPrice;
    public String status;

    public Booking(Long id, Long venueId, String venueName, String userName, String bookingDate, Integer hours, Double totalPrice) {
        this.id = id;
        this.venueId = venueId;
        this.venueName = venueName;
        this.userName = userName;
        this.bookingDate = bookingDate;
        this.hours = hours;
        this.totalPrice = totalPrice;
        this.status = "PENDING";
    }
}

class Payment {
    public Long id;
    public Long bookingId;
    public String userName;
    public Double amount;
    public String paymentMethod;
    public String status;
    public String paymentDate;

    public Payment(Long id, Long bookingId, String userName, Double amount, String paymentMethod) {
        this.id = id;
        this.bookingId = bookingId;
        this.userName = userName;
        this.amount = amount;
        this.paymentMethod = paymentMethod;
        this.status = "COMPLETED";
        this.paymentDate = LocalDate.now().toString();
    }
}

public class VenueBookingApp {
    static List<Venue> venues = new ArrayList<>();
    static List<Booking> bookings = new ArrayList<>();
    static List<Payment> payments = new ArrayList<>();
    static Long venueIdCounter = 1L;
    static Long bookingIdCounter = 1L;
    static Long paymentIdCounter = 1L;

    public static void main(String[] args) {
        try {
            addSampleVenues();

            HttpServer server = HttpServer.create(new InetSocketAddress("localhost", 8080), 0);
            System.out.println("Server created successfully");

            server.createContext("/", exchange -> {
                try {
                    if (exchange.getRequestMethod().equals("GET")) {
                        String html = getHTML();
                        exchange.getResponseHeaders().set("Content-Type", "text/html; charset=UTF-8");
                        exchange.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
                        exchange.sendResponseHeaders(200, html.getBytes().length);
                        OutputStream os = exchange.getResponseBody();
                        os.write(html.getBytes());
                        os.close();
                        System.out.println("Home page served");
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });

            server.createContext("/api/venues", exchange -> {
                try {
                    exchange.getResponseHeaders().set("Content-Type", "application/json");
                    exchange.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
                    
                    if (exchange.getRequestMethod().equals("GET")) {
                        String query = exchange.getRequestURI().getQuery();
                        List<Venue> result = new ArrayList<>(venues);
                        
                        if (query != null && query.contains("location=")) {
                            String location = query.split("=")[1].replace("%20", " ");
                            result = venues.stream()
                                .filter(v -> v.location.equalsIgnoreCase(location))
                                .collect(Collectors.toList());
                        }
                        
                        String json = venueListToJson(result);
                        exchange.sendResponseHeaders(200, json.getBytes().length);
                        exchange.getResponseBody().write(json.getBytes());
                        exchange.getResponseBody().close();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });

            server.createContext("/api/bookings", exchange -> {
                try {
                    exchange.getResponseHeaders().set("Content-Type", "application/json");
                    exchange.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
                    
                    if (exchange.getRequestMethod().equals("POST")) {
                        InputStreamReader isr = new InputStreamReader(exchange.getRequestBody());
                        BufferedReader br = new BufferedReader(isr);
                        StringBuilder sb = new StringBuilder();
                        String line;
                        while ((line = br.readLine()) != null) {
                            sb.append(line);
                        }
                        String body = sb.toString();
                        
                        Long venueId = Long.parseLong(extractValue(body, "venueId"));
                        String userName = extractValue(body, "userName");
                        String bookingDate = extractValue(body, "bookingDate");
                        Integer hours = Integer.parseInt(extractValue(body, "hours"));
                        
                        Venue venue = venues.stream().filter(v -> v.id.equals(venueId)).findFirst().orElse(null);
                        if (venue != null) {
                            Double totalPrice = venue.pricePerHour * hours;
                            Booking booking = new Booking(bookingIdCounter++, venueId, venue.name, userName, bookingDate, hours, totalPrice);
                            bookings.add(booking);
                            
                            String json = bookingToJson(booking);
                            exchange.sendResponseHeaders(200, json.getBytes().length);
                            exchange.getResponseBody().write(json.getBytes());
                        }
                        exchange.getResponseBody().close();
                    } else if (exchange.getRequestMethod().equals("GET")) {
                        String json = bookingListToJson(bookings);
                        exchange.sendResponseHeaders(200, json.getBytes().length);
                        exchange.getResponseBody().write(json.getBytes());
                        exchange.getResponseBody().close();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });

            server.createContext("/api/payments", exchange -> {
                try {
                    exchange.getResponseHeaders().set("Content-Type", "application/json");
                    exchange.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
                    
                    if (exchange.getRequestMethod().equals("POST")) {
                        InputStreamReader isr = new InputStreamReader(exchange.getRequestBody());
                        BufferedReader br = new BufferedReader(isr);
                        StringBuilder sb = new StringBuilder();
                        String line;
                        while ((line = br.readLine()) != null) {
                            sb.append(line);
                        }
                        String body = sb.toString();
                        
                        Long bookingId = Long.parseLong(extractValue(body, "bookingId"));
                        String userName = extractValue(body, "userName");
                        String paymentMethod = extractValue(body, "paymentMethod");
                        
                        Booking booking = bookings.stream().filter(b -> b.id.equals(bookingId)).findFirst().orElse(null);
                        if (booking != null) {
                            booking.status = "CONFIRMED";
                            Payment payment = new Payment(paymentIdCounter++, bookingId, userName, booking.totalPrice, paymentMethod);
                            payments.add(payment);
                            
                            String json = paymentToJson(payment);
                            exchange.sendResponseHeaders(200, json.getBytes().length);
                            exchange.getResponseBody().write(json.getBytes());
                        }
                        exchange.getResponseBody().close();
                    } else if (exchange.getRequestMethod().equals("GET")) {
                        String json = paymentListToJson(payments);
                        exchange.sendResponseHeaders(200, json.getBytes().length);
                        exchange.getResponseBody().write(json.getBytes());
                        exchange.getResponseBody().close();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });

            server.setExecutor(null);
            server.start();
            System.out.println("\n========================================");
            System.out.println("✅ Venue Booking Server Started!");
            System.out.println("========================================");
            System.out.println("Open your browser and go to:");
            System.out.println("👉 http://localhost:8080");
            System.out.println("========================================\n");
        } catch (Exception e) {
            System.err.println("Error starting server: " + e.getMessage());
            e.printStackTrace();
        }
    }

    static void addSampleVenues() {
        venues.add(new Venue(venueIdCounter++, "Grand Hall", "New York", "Large event space", 500.0, 500));
        venues.add(new Venue(venueIdCounter++, "Riverside Resort", "California", "Luxury venue", 750.0, 300));
        venues.add(new Venue(venueIdCounter++, "City Center", "New York", "Downtown venue", 400.0, 250));
        venues.add(new Venue(venueIdCounter++, "Beach Paradise", "Florida", "Beachfront venue", 600.0, 200));
        venues.add(new Venue(venueIdCounter++, "Mountain Lodge", "Colorado", "Rustic lodge", 550.0, 150));
    }

    static String venueListToJson(List<Venue> list) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < list.size(); i++) {
            Venue v = list.get(i);
            sb.append("{\"id\":").append(v.id).append(",\"name\":\"").append(v.name)
              .append("\",\"location\":\"").append(v.location).append("\",\"description\":\"")
              .append(v.description).append("\",\"pricePerHour\":").append(v.pricePerHour)
              .append(",\"capacity\":").append(v.capacity).append("}");
            if (i < list.size() - 1) sb.append(",");
        }
        sb.append("]");
        return sb.toString();
    }

    static String bookingToJson(Booking b) {
        return "{\"id\":" + b.id + ",\"venueName\":\"" + b.venueName + "\",\"userName\":\"" + b.userName + "\",\"bookingDate\":\"" + b.bookingDate + "\",\"hours\":" + b.hours + ",\"totalPrice\":" + b.totalPrice + ",\"status\":\"" + b.status + "\"}";
    }

    static String bookingListToJson(List<Booking> list) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < list.size(); i++) {
            sb.append(bookingToJson(list.get(i)));
            if (i < list.size() - 1) sb.append(",");
        }
        sb.append("]");
        return sb.toString();
    }

    static String paymentToJson(Payment p) {
        return "{\"id\":" + p.id + ",\"bookingId\":" + p.bookingId + ",\"userName\":\"" + p.userName + "\",\"amount\":" + p.amount + ",\"paymentMethod\":\"" + p.paymentMethod + "\",\"status\":\"" + p.status + "\",\"paymentDate\":\"" + p.paymentDate + "\"}";
    }

    static String paymentListToJson(List<Payment> list) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < list.size(); i++) {
            sb.append(paymentToJson(list.get(i)));
            if (i < list.size() - 1) sb.append(",");
        }
        sb.append("]");
        return sb.toString();
    }

    static String extractValue(String json, String key) {
        try {
            int start = json.indexOf("\"" + key + "\"");
            if (start == -1) return "";
            start = json.indexOf(":", start) + 1;
            String sub = json.substring(start).trim();
            if (sub.startsWith("\"")) {
                return sub.substring(1, sub.indexOf("\"", 1));
            } else {
                int end = sub.indexOf(",");
                if (end == -1) end = sub.indexOf("}");
                return sub.substring(0, end).trim();
            }
        } catch (Exception e) {
            return "";
        }
    }

    static String getHTML() {
        return "<!DOCTYPE html><html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width,initial-scale=1'><title>Venue Booking</title><style>*{margin:0;padding:0;box-sizing:border-box}body{font-family:Arial,sans-serif;background:linear-gradient(135deg,#667eea 0%,#764ba2 100%);min-height:100vh;padding:20px}.container{max-width:1200px;margin:0 auto}header{background:white;padding:30px;border-radius:10px;text-align:center;box-shadow:0 4px 6px rgba(0,0,0,0.1);margin-bottom:20px}header h1{color:#667eea;font-size:2em;margin-bottom:10px}header p{color:#666}.nav-tabs{display:flex;gap:10px;justify-content:center;margin-bottom:20px;flex-wrap:wrap}.nav-tabs button{padding:10px 20px;border:none;background:#667eea;color:white;border-radius:5px;cursor:pointer;font-size:1em;transition:all 0.3s}.nav-tabs button:hover,.nav-tabs button.active{background:#764ba2;transform:translateY(-2px)}.content{background:white;padding:30px;border-radius:10px;box-shadow:0 4px 6px rgba(0,0,0,0.1)}.section{display:none}.section.active{display:block}h2{color:#667eea;margin-bottom:20px;border-bottom:2px solid #667eea;padding-bottom:10px}h3{color:#333;margin:20px 0 15px 0}.search-box{display:flex;gap:10px;margin-bottom:30px;flex-wrap:wrap}.search-box input,.search-box button{padding:12px 15px;border:1px solid #ddd;border-radius:5px;font-size:1em}.search-box input{flex:1;min-width:200px}.search-box button{background:#667eea;color:white;cursor:pointer;transition:all 0.3s;min-width:100px}.search-box button:hover{background:#764ba2}.location-list{display:flex;gap:10px;margin:20px 0;flex-wrap:wrap}.location-btn{padding:8px 16px;background:#e9ecef;border:1px solid #ddd;border-radius:5px;cursor:pointer;transition:all 0.3s;font-size:0.9em}.location-btn:hover{background:#667eea;color:white;border-color:#667eea}.venue-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:20px;margin:20px 0}.venue-card{background:#f8f9fa;border:1px solid #ddd;border-radius:8px;padding:20px;box-shadow:0 2px 8px rgba(0,0,0,0.1);transition:all 0.3s}.venue-card:hover{transform:translateY(-5px);box-shadow:0 4px 12px rgba(0,0,0,0.15);border-color:#667eea}.venue-card h3{color:#667eea;margin-bottom:10px}.venue-card p{color:#666;margin:8px 0;font-size:0.95em}.price{font-size:1.4em;color:#764ba2;font-weight:bold;margin:15px 0}.venue-card button{width:100%;padding:10px;background:#667eea;color:white;border:none;border-radius:5px;cursor:pointer;transition:all 0.3s;font-weight:bold}.venue-card button:hover{background:#764ba2;transform:translateY(-2px)}.form-group{margin-bottom:15px}.form-group label{display:block;margin-bottom:5px;color:#333;font-weight:bold}.form-group input,.form-group select{width:100%;padding:10px;border:1px solid #ddd;border-radius:5px;font-size:1em}.form-group button{width:100%;padding:12px;background:#667eea;color:white;border:none;border-radius:5px;cursor:pointer;font-size:1em;font-weight:bold;transition:all 0.3s}.form-group button:hover{background:#764ba2;transform:translateY(-2px)}.booking-form,.payment-form{background:#f8f9fa;padding:20px;border-radius:8px;margin-bottom:20px;border:1px solid #ddd}table{width:100%;border-collapse:collapse;margin:20px 0;font-size:0.95em}table th{background:#667eea;color:white;padding:12px;text-align:left}table td{padding:12px;border-bottom:1px solid #ddd}table tr:hover{background:#f0f0f0}.status-badge{padding:5px 10px;border-radius:5px;font-weight:bold;font-size:0.85em;display:inline-block}.status-pending{background:#fff3cd;color:#856404}.status-confirmed,.status-completed{background:#d4edda;color:#155724}.success-message{background:#d4edda;color:#155724;padding:15px;border-radius:5px;margin-bottom:20px;display:none;border-left:4px solid #155724;font-weight:bold}.success-message.show{display:block}@media(max-width:768px){.venue-grid{grid-template-columns:1fr}.search-box{flex-direction:column}.search-box button{width:100%}table{font-size:0.85em}table th,table td{padding:8px}}</style></head><body><div class='container'><header><h1>🏛 Venue Booking System</h1><p>Find, Schedule, and Pay for Perfect Venues</p></header><div class='nav-tabs'><button class='active' onclick=\"showTab('venues')\">🔍 Find Venues</button><button onclick=\"showTab('bookings')\">📅 My Bookings</button><button onclick=\"showTab('payments')\">💳 Payments</button></div><div class='content'><div id='venues' class='section active'><h2>🏢 Find Your Perfect Venue</h2><div class='search-box'><input type='text' id='searchLocation' placeholder='Search by location (e.g., New York)'><button onclick='searchVenues()'>🔍 Search</button><button onclick='loadVenues()'>📍 Show All</button></div><div class='location-list' id='locationList'></div><div class='venue-grid' id='venueGrid'></div></div><div id='bookings' class='section'><h2>📅 Create & Manage Bookings</h2><div class='success-message' id='bookingMsg'></div><div class='booking-form'><h3>📝 New Booking</h3><div class='form-group'><label>Your Name:</label><input type='text' id='userName' placeholder='Enter your full name'></div><div class='form-group'><label>Select Venue:</label><select id='venueSelect'><option value=''>Choose a venue...</option></select></div><div class='form-group'><label>Booking Date:</label><input type='date' id='bookingDate'></div><div class='form-group'><label>Duration (Hours):</label><input type='number' id='hours' min='1' max='24' value='4'></div><div class='form-group'><button onclick='createBooking()'>✅ Book Now</button></div></div><h3>📋 All Bookings</h3><table id='bookingTable'><thead><tr><th>ID</th><th>Venue</th><th>Customer</th><th>Date</th><th>Hours</th><th>Price</th><th>Status</th></tr></thead><tbody id='bookingList'><tr><td colspan='7' style='text-align:center;color:#999'>No bookings yet</td></tr></tbody></table></div><div id='payments' class='section'><h2>💳 Payment Processing</h2><div class='success-message' id='paymentMsg'></div><div class='payment-form'><h3>💰 Process Payment</h3><div class='form-group'><label>Your Name:</label><input type='text' id='paymentName' placeholder='Enter your full name'></div><div class='form-group'><label>Booking ID:</label><input type='number' id='bookingId' placeholder='Enter booking ID'></div><div class='form-group'><label>Payment Method:</label><select id='paymentMethod'><option value='CREDIT_CARD'>💳 Credit Card</option><option value='DEBIT_CARD'>🏧 Debit Card</option><option value='PAYPAL'>🅿 PayPal</option><option value='BANK'>🏦 Bank Transfer</option></select></div><div class='form-group'><button onclick='processPayment()'>✅ Pay Now</button></div></div><h3>📊 Payment History</h3><table id='paymentTable'><thead><tr><th>ID</th><th>Booking</th><th>Customer</th><th>Amount</th><th>Method</th><th>Date</th><th>Status</th></tr></thead><tbody id='paymentList'><tr><td colspan='7' style='text-align:center;color:#999'>No payments yet</td></tr></tbody></table></div></div></div></div><script>let allVenues=[];function showTab(id){document.querySelectorAll('.section').forEach(s=>s.classList.remove('active'));document.getElementById(id).classList.add('active');document.querySelectorAll('.nav-tabs button').forEach(b=>b.classList.remove('active'));event.target.classList.add('active');if(id==='bookings')loadBookings();if(id==='payments')loadPayments()}function loadVenues(){fetch('/api/venues').then(r=>r.json()).then(data=>{allVenues=data;displayVenues(data);updateLocationFilter(data);updateVenueSelect(data)}).catch(e=>alert('Error loading venues: '+e))}function displayVenues(data){const grid=document.getElementById('venueGrid');if(data.length===0){grid.innerHTML='<p style=\"color:#999\">No venues found</p>';return}grid.innerHTML=data.map(v=><div class='venue-card'><h3>${v.name}</h3><p><strong>📍</strong> ${v.location}</p><p>${v.description}</p><p><strong>👥</strong> Capacity: ${v.capacity}</p><div class='price'>$${v.pricePerHour}/hr</div><button onclick=\"selectVenue(${v.id})\">Book Now</button></div>).join('')}function updateLocationFilter(data){const locs=[...new Set(data.map(v=>v.location))];document.getElementById('locationList').innerHTML=locs.map(l=><button class='location-btn' onclick=\"document.getElementById('searchLocation').value='${l}';searchVenues()\">${l}</button>).join('')}function updateVenueSelect(data){const sel=document.getElementById('venueSelect');sel.innerHTML='<option value=\"\">Choose a venue...</option>'+data.map(v=><option value=\"${v.id}\">${v.name} ($${v.pricePerHour}/hr)</option>).join('')}function selectVenue(id){document.getElementById('venueSelect').value=id;showTab('bookings')}function searchVenues(){const loc=document.getElementById('searchLocation').value;if(!loc){alert('Please enter a location');return}fetch('/api/venues?location='+encodeURIComponent(loc)).then(r=>r.json()).then(displayVenues).catch(e=>alert('Error: '+e))}function createBooking(){const vId=document.getElementById('venueSelect').value;const name=document.getElementById('userName').value;const date=document.getElementById('bookingDate').value;const hrs=document.getElementById('hours').value;if(!vId||!name||!date||!hrs){alert('Please fill all fields');return}fetch('/api/bookings',{method:'POST',body:JSON.stringify({venueId:parseInt(vId),userName:name,bookingDate:date,hours:parseInt(hrs)})}).then(r=>r.json()).then(b=>{showMsg('bookingMsg','✓ Booking '+b.id+' created! Total: $'+b.totalPrice);document.getElementById('userName').value='';document.getElementById('hours').value='4';loadBookings()}).catch(e=>alert('Error: '+e))}function loadBookings(){fetch('/api/bookings').then(r=>r.json()).then(data=>{const body=document.getElementById('bookingList');body.innerHTML=data.length?data.map(b=><tr><td>${b.id}</td><td>${b.venueName}</td><td>${b.userName}</td><td>${b.bookingDate}</td><td>${b.hours}</td><td>$${b.totalPrice}</td><td><span class='status-badge status-${b.status.toLowerCase()}'>${b.status}</span></td></tr>).join(''):'<tr><td colspan=7 style=\"text-align:center;color:#999\">No bookings yet</td></tr>'}).catch(e=>alert('Error: '+e))}function processPayment(){const name=document.getElementById('paymentName').value;const bId=document.getElementById('bookingId').value;const method=document.getElementById('paymentMethod').value;if(!name||!bId||!method){alert('Please fill all fields');return}fetch('/api/payments',{method:'POST',body:JSON.stringify({userName:name,bookingId:parseInt(bId),paymentMethod:method})}).then(r=>r.json()).then(p=>{showMsg('paymentMsg','✓ Payment $'+p.amount+' processed successfully!');document.getElementById('paymentName').value='';document.getElementById('bookingId').value='';loadPayments();loadBookings()}).catch(e=>alert('Error: '+e))}function loadPayments(){fetch('/api/payments').then(r=>r.json()).then(data=>{const body=document.getElementById('paymentList');body.innerHTML=data.length?data.map(p=><tr><td>${p.id}</td><td>${p.bookingId}</td><td>${p.userName}</td><td>$${p.amount}</td><td>${p.paymentMethod}</td><td>${p.paymentDate}</td><td><span class='status-badge status-${p.status.toLowerCase()}'>${p.status}</span></td></tr>).join(''):'<tr><td colspan=7 style=\"text-align:center;color:#999\">No payments yet</td></tr>'}).catch(e=>alert('Error: '+e))}function showMsg(id,text){const m=document.getElementById(id);m.textContent=text;m.classList.add('show');setTimeout(()=>m.classList.remove('show'),4000)}window.onload=loadVenues</script></body></html>";
    }
}# Venue-Booking-System
A simple Java-based venue booking system that lets users browse venues, make bookings, and process payments using REST APIs and a basic web interface.
