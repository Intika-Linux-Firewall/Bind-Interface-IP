# BinD

<div class="article-body">

<h3>How to bind a process to an IP address using LD_PRELOAD (on Linux)</h3>

<p>Sometimes a peice of software does not provide a configuration option to specify the IP address to bind to on a multi-ip server (including GRE tunnels!). This could be due to immaturity of the software (7 Days to Die) or that it is no longer under development (utdns). It is possible to work around this on Linux without modifying the source (if available) by using a injected (preload) dynamic library to detour the the socket functions and add the required functionality.</p>

<h3>Compiling the LD_PRELOAD shim</h3>

<p>The shim we are going to be using in this example was written by <a href="http://www.ryde.net/">Daniel Ryde</a>. With thanks to Daniel Lange for <a href="http://daniel-lange.com/archives/53-Binding-applications-to-a-specific-IP.html">his guide</a> and locally hosted copy of the code.</p>

<h4>Prerequisites</h4>

<ol>
<li><p>Install the GCC compiler, essential libraries and development headers.</p>

<p><strong><em>On Debian / Ubuntu:</em></strong></p>

<pre><code>apt-get install build-essential
</code></pre>

<p><strong><em>On CentOS / Redhat / Fedora:</em></strong></p>

<pre><code>yum groupinstall "Development Tools"
</code></pre></li>
<li><p>If compiling on a 64bit operating system for a 32bit application install the 32bit libraries and developemnt headers</p>

<p><strong><em>On Debian / Ubuntu:</em></strong></p>

<pre><code>apt-get install g++-multilib libc6-dev-i386
</code></pre>

<p><strong><em>On CentOS 6.x / Redhat / Fedora:</em></strong></p>

<pre><code>yum reinstall glibc.i686 glibc-devel.i686
</code></pre>

<p><strong><em>On CentOS 4.x, 5.x / Redhat / Fedora:</em></strong></p>

<pre><code>yum reinstall glibc.i386 glibc-devel.i386
</code></pre></li>
</ol>

<h4>Compiling the Shim with GCC</h4>

<ol>
<li><p>Get a copy of the shim code. This code can be obtained from:</p>

<ul>
<li><a href="https://www.x4b.net/files/bind.c.txt">https://www.x4b.net/files/bind.c.txt</a></li>
<li><a href="http://www.ryde.net/code/bind.c.txt">http://www.ryde.net/code/bind.c.txt</a></li>
<li><a href="http://daniel-lange.com/software/bind.c">http://daniel-lange.com/software/bind.c</a></li>
</ul>

<p><br>
To download this onto your server using wget:</p>

<pre><code>wget https://www.x4b.net/files/bind.c.txt -O bind.c
</code></pre></li>
<li><p>Compile the shim using GCC.</p>

<p><strong><em>For the architecture of the compiling server:</em></strong></p>

<pre><code>gcc -nostartfiles -fpic -shared bind.c -o bind.so -ldl -D_GNU_SOURCE
</code></pre>

<p><strong><em>For a 32bit application, compiled on a 64bit server:</em></strong></p>

<pre><code>gcc -nostartfiles -fpic -shared bind.c -o bind32.so -ldl -D_GNU_SOURCE -m32
</code></pre></li>
<li><p>Strip excess symbols from the file. If crosscompiling for a 32bit architecture the shared object should be bind32.so not bind.so.</p>

<pre><code>strip bind.so
</code></pre></li>
<li><p>Move the shared object to <code>/usr/lib/</code></p>

<pre><code>cp -i bind.so /usr/lib/
</code></pre></li>
</ol>

<h3>Using the compiled shim</h3>

<p>To use the compiled shim with your application or service it needs to be "preloaded" into the application. This is accomplished with the environment variable <code>LD_PRELOAD</code> which should be the path to the shared object compiled. An environment variable <code>BIND_ADDR</code> is provided to specify the IP address for the service to bind to.</p>

<pre><code>BIND_ADDR="X.X.X.X" LD_PRELOAD=/usr/lib/bind.so [command to run]
</code></pre>

<p>The command to run can be the daemon of the application, or a utility such as <code>start-stop-daemon</code> (providing the utility passes on environment variables).</p>

<p>With X4B GRE Tunnels the <code>BIND_ADDR</code> is most likely the IP address of your backend's side of the GRE tunnel. This can be obtained from <code>ifconfig</code> or your service panel (Tunnel Information).</p>

<pre><code># ifconfig gre2
gre2 Link encap:UNSPEC HWaddr 6C-3D-CF-72-00-00-B0-B6-00-00-00-00-00-00-00-00
          inet addr:10.17.24.14 P-t-P:10.17.24.14 Mask:255.255.255.252
          UP POINTOPOINT RUNNING NOARP MTU:1180 Metric:1
          RX packets:30826 errors:0 dropped:0 overruns:0 frame:0
          TX packets:41206 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2782629 (2.6 MiB) TX bytes:6674435 (6.3 MiB)
</code></pre>

<p>For an example of this shim in use see the <a href="/kb/7DtDDDoSProtection">7 Days to Die DDoS protection tutorial</a>.</p>

<h3>How this works</h3>

<p><strong>Warning: Technical Content ahead, feel free to skip</strong></p>

<p>These days many systems are multi-homed in the sense that they have more than one IP address bound at the same time. These IP addresses could be additional IPs added to your server for additional HTTPS sites, services (where a specific port is required) or connected to different networks.</p>

<p>The kernel chooses an outgoing IP from the records matcing the destination from the routing table. Multiple routes are sorted based on the value of the metric:</p>

<p><img src="http://img.x4b.org/13-08-2014/22_44_13_new_54_Notepad_.png" alt="Output of Route"></p>

<p>It is possible to alter the metric and make the kernel router prefer the desired interface above others. However this will affect all applications and services on the server. This includes private services such as SSH, and makes it difficult to use the additional IP addresses on the server.</p>

<p>Applications / Services are able to choose to bind to a specific interface, and often provide the option via a configuration file to choose the network interface or IP address to bind to. Unfortunately not all applications and services provide this functionality, and hence it becomes necessary to add this functionality after the fact. Often this has to be done without knowledge of the underlying software's source code, either because it is closed source or where compilation is difficult.</p>

<p>To do this a LD_PRELOAD library is made to modify bind and connect to use a specific localaddress, regardless of the parameters given to them by the application or service. In doing such the application or service is hence bound to the localaddress for listening and connecting sockets.</p>
