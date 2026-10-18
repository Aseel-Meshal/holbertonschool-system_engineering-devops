# Web infrastructure design

Each answer file contains the URL of its diagram. The diagrams are in `images/`.

## 0. Simple web stack

A user enters `www.foobar.com` in a browser. DNS returns `8.8.8.8` from the `www` A record in the `foobar.com` domain. The browser sends an HTTP request to that server over TCP/IP. Nginx serves static files or forwards dynamic requests to the application server. The application executes the code base and queries MySQL when needed. The response returns through Nginx to the browser.

- A server is a physical or virtual computer, or a program, that provides services to clients. Here it is one computer hosting all the components.
- The domain name provides a readable name instead of requiring users to remember an IP address.
- An A record maps a hostname to an IPv4 address.
- The web server handles HTTP requests, static files and forwarding requests to the application server.
- The application server runs the application logic using the application files.
- MySQL stores and retrieves structured application data.
- The server communicates with the user's computer using HTTP over TCP/IP.
- SPOF: the server is a single point of failure. A failure of its hardware or a required component can make the website unavailable.
- Maintenance or restarting a required service can cause downtime because there is no alternate server.
- One machine has limited CPU, memory, storage and network capacity, so it cannot handle unlimited traffic.

## 1. Distributed web infrastructure

The user resolves `www.foobar.com` to the load balancer's public IP. There are three servers: one HAProxy server and two servers that each contain Nginx, an application server, the application files and MySQL.

- HAProxy distributes requests so one application server does not receive all the traffic. It uses round robin: each healthy backend receives the next request in turn. Health checks remove failed backends from rotation.
- A second application backend adds capacity and allows the other backend to serve requests during maintenance or a failure.
- Both application backends serve requests, making the backend setup active-active. In active-passive, a standby backend takes over only when the active one fails. HAProxy itself has no standby in this design.
- MySQL uses primary-replica replication. The primary records changes in its binary log; the replica receives and applies them. Asynchronous replication can lag.
- Both applications send writes to the primary. Reads that tolerate lag can use the replica; reads requiring current data use the primary. The replica is read-only for application traffic.
- SPOF: HAProxy is the only entry point, and the MySQL primary is the only node accepting writes. Losing the primary requires promotion of a replica and application reconfiguration before writes resume.
- Security issues: no firewalls restrict access, and HTTP traffic is unencrypted because there is no HTTPS.
- There is no monitoring to detect failures, performance problems or unusual activity.

## 2. Secured and monitored web infrastructure

This keeps the three-server design from task 1 and adds three host firewalls, one certificate for `www.foobar.com` installed at HAProxy, and one monitoring client on each server.

- The load balancer firewall allows public HTTPS on TCP 443. Each backend firewall allows HTTP only from HAProxy. MySQL access is limited to application and replication traffic from the required hosts. Administration is limited to trusted management addresses.
- Firewalls filter network traffic according to rules and block unwanted connections.
- HTTPS uses TLS to encrypt browser-to-load-balancer traffic, protect it against tampering and authenticate the website using its certificate.
- Monitoring detects availability problems, errors and resource limits. Each agent collects host metrics, service metrics and logs, then sends them over HTTPS to a remote monitoring service such as Sumo Logic. Dashboards and alerts make problems visible.
- To monitor Nginx QPS (queries or requests per second), collect a request counter or access logs. Calculate the change in request count divided by the time interval in seconds. Graph the rate and alert on abnormal changes. Nginx `stub_status` can expose the cumulative requests counter to an authorized collector.
- TLS terminates at HAProxy, so the connection from HAProxy to Nginx is plain HTTP. An attacker with access to that network could read or alter it. Encrypting backend connections would address this gap.
- Only the MySQL primary accepts writes. It is a write bottleneck and a single point of failure for writes; a replica does not automatically replace it.
- Combining web, application and database components on the same machines causes competition for resources, couples failures and deployments, and makes independent scaling harder.
- The single load balancer remains a SPOF.

## 3. Scale up

Starting from the three servers in task 2, add one server for the separated tiers and one additional HAProxy server. The result is five servers: two load balancers, one web server, one application server and one database server.

- The two HAProxy servers form an active-passive cluster using Keepalived/VRRP and a shared virtual IP. DNS points to this VIP. The standby takes ownership if the active load balancer fails, removing the single load-balancer dependency. Health checks must detect HAProxy failure as well as host failure. The certificate and matching configuration are installed on both load balancers.
- The added tier server allows Nginx, the application server with its code base, and MySQL to run on separate machines. Each tier can be maintained, sized and expanded independently, and they no longer compete on one machine for the same resources.
- Traffic flows from HAProxy to Nginx, then to the application server, then to MySQL. Replies return along the request path. Firewalls restrict each connection to its required source and port, and each host sends monitoring data to the remote service.
- This minimal split uses one MySQL server rather than retaining the earlier replica. The web, application and database tiers each still have a SPOF. More replicas would be needed for full high availability. Backend HTTP also remains unencrypted after TLS termination.

## Terms

- LAMP: Linux, Apache, MySQL, PHP/Python/Perl. These diagrams use Nginx instead of Apache, as required.
- SPOF: Single Point Of Failure.
- QPS: Queries Per Second; for the web server, the HTTP request rate.

## Submission

Paste each GitHub answer-file URL into its corresponding task URL box. Request the manual QA review after submitting all four tasks. Practice drawing and explaining each design without notes for the whiteboarding session.
