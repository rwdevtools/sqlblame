# Attach SQL Blame

SQL Blame normally joins your application when the IDE starts it. **Attach SQL Blame...** covers the other case: an
application that is already running outside the IDE, such as `java -jar app.jar` in a terminal, a local Tomcat, or a
Spring Boot service in Docker or on a dev box. You don't restart it and don't copy any jar. The IDE loads the agent
into the running JVM, and from then on its queries appear in the SQL Blame tool window like those of any other run.

There are two ways in:

| | Local | Remote |
|---|---|---|
| Where the application runs | This computer, under your OS user | A container, another machine, a port forward |
| What it needs | Nothing | A JDWP debug port |
| How the agent gets in | The JDK's Attach API, as IntelliJ's "Attach to Process..." | Through the debug port, as Evaluate Expression does |
| Label in the tool window | `[attached]` | `[remote]` |

Attaching needs a license or an active trial, like capturing a run started from the IDE.

## Opening the dialog

**Attach SQL Blame...** is in the Run menu, right after "Attach to Process...". The same action is the plug button in
the toolbars of the Queries and Processes tabs, and a link in the empty Queries tab. Find Action finds it by name.
The dialog opens on the tab you used last.

## Local applications

The Local tab lists the Java applications running under your user: main class (or jar, or application server), pid,
Java version, how long it has been running and whether it can be attached. The IDE itself, Gradle and Maven daemons
and IntelliJ's helper JVMs are hidden, and **Show all JVMs** shows them. Applications SQL Blame already captures are
greyed out as Captured, and those of another OS user as Other user. Type in the search field to filter by name, jar or
pid. The list refreshes when the dialog opens and with the refresh button.

Select the application and click **Attach** or double-click it. The attach takes about half a second. The run shows
up in the Processes tab as "Attaching...", then as `[attached] LibraryApplication · pid 18244 · Java 21`, and a
balloon offers **Show Queries**.

The application needs no flag, with one exception on Java 21 and later (see [JEP 451](#java-21-and-later-jep-451)).

## Remote applications

Remote attach works through a JDWP debug port. The application must have been started with one, for example:

```text
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

On Java 8 write `address=5005`. Java 9 and later need `*:5005` to accept connections from outside the container or
machine. For a container, publish the port (`docker run -p 5005:5005 ...`). For Kubernetes, forward it
(`kubectl port-forward pod/orders-7d9f 5005:5005`) and attach to `localhost:5005`.

The Remote tab shows both options with a **Copy** button: the debug port option for every Java version and
`-XX:+EnableDynamicAgentLoading` for Java 21 and later (see [JEP 451](#java-21-and-later-jep-451)).

On the Remote tab, enter the host and the debug port and click **Attach**. **Test Connection** checks that a JVM
answers on the port and shows its Java version, without loading anything. Each target you attach to is kept in the
saved targets on the left, named `host:port` until you rename it. No password is ever stored.

**Advanced** holds two optional ports for networks where the agent cannot connect back to the IDE. The JMX port is
one the application already opened, without authentication. The agent port is a port the agent may listen on, as
this computer reaches it: for a container started with `-p 9455:9455`, enter 9455.

The first remote attach asks for confirmation: remote attach runs code in the application through its debugger
connection, so use it on development and test environments only. You can turn the question off there or in the
settings.

### What happens during a remote attach

The IDE opens a debugger connection and waits for one application thread to call into the JDK, which happens as
soon as the application does any work. It stops that one thread, removes its temporary breakpoint, and through that
thread writes the agent jar into the application's temp directory (`sqlblame-agent-<hash>.jar`, reused by the next
attach of the same version) and loads it. Then the thread continues. No other thread is stopped.

An application that sits idle gives the IDE no thread to use. After 3 s the progress text asks you to use the
application once, for example to send a request. After 30 s the attach gives up with a message and **Try Again**. A
remote attach takes about one second once a thread turns up.

Unless the events travel over the debug port itself (the JDWP pump below), the IDE then closes its debugger
connection, so the port is free for IntelliJ's own debugger again.

## How the events reach the IDE

All measurement happens inside the application, by the same agent code as in a run started from the IDE. Only the
delivery differs. SQL Blame picks the first way that works and shows it in the tooltip of the run's Processes row.

| Transport | Used for | When |
|---|---|---|
| Local connection | Local | Always: the agent connects to the IDE on `localhost`, as in runs started from the IDE |
| Connection back to the IDE | Remote | Tried first. The agent connects to the IDE, from Docker on this computer through `host.docker.internal`, from another machine to this computer's address on the debugger connection |
| Agent port | Remote | When you gave an agent port: the IDE connects to the agent |
| JMX | Remote | When you gave a JMX port: the IDE reads the events through an MBean of the agent |
| Debug port (JDWP pump) | Remote | When nothing else works: a thread of the agent, `sqlblame-jdwp-pump`, stops briefly about every 300 ms and the IDE collects the events through the debugger connection |

The IDE gives each transport up to about 10 s and then loads the agent again with the next one, so a firewall on the
way makes the attach slower but does not stop it. For the connection back, the IDE opens a second port on all of its
network interfaces at the first remote attach, also for a container on this computer. Nothing is accepted on it
until the agent presents the run's token, and every other transport needs that token too.

The debug-port transport keeps a debugger connected for the whole run. Its breakpoint stops only the
`sqlblame-jdwp-pump` thread, never an application thread, but while it is active no other debugger can use the port.
Detach frees it.

If the connection breaks, the run shows "Connection lost, reconnecting..." and continues when the transport is
back. After 60 s without a connection the agent stops capturing and the run ends.

## Attached runs in the tool window

Attached runs work like any other run: grouping, call paths, rules, Explain Plan, Export Run..., Rename Run... and the
run history. The Processes row shows the state (Attaching..., Capturing, Detached, Connection lost,
reconnecting..., Ended), and its context menu adds two actions. **Detach** turns capture off and closes the
connection while the application keeps running. The JVM cannot unload classes, so the agent's hooks stay in it and
only pass calls through. **Attach Again** turns capture on again, as a new run, and reuses the agent already in the
JVM.

Times shown for a remote run are corrected by the difference between the two machines' clocks. Durations are measured
in the application and never touched.

## What is captured after an attach

Queries on connections the application gets after the attach. With a connection pool (HikariCP and the others) that
is every connection borrowed from the pool after the attach, so in practice all queries from the next request on.
A connection that a thread borrowed earlier and still holds, such as a long batch job's, stays invisible until it is
returned and borrowed again. The Queries tab says this once for each attached run.

The caller of a query is the first stack frame in the application's packages. By default the attach passes the packages
of the open project, and **Change...** in the dialog sets others for one attach. With that setting off, the
agent takes them from the application's main class or jar.

The numbers are the same as in a run started with the agent: in the author's measurement over 70 JVMs every
transport gave the same query counts, no dropped events and the same rule findings, with times
within 1.6% of an agent started with the JVM (3.4% for the debug-port transport). The first 10 s after an attach run
2-5% slower while the JVM compiles the agent's code. SB004 (slow query) still counts those executions, since the
difference is far below its 100 ms threshold.

If the IDE cannot read events as fast as the application produces them, the agent drops events instead of slowing
the application. The run then says how many in the Processes tab and in a line above the Queries list, because
counts and rule results are lower than they should be.

## Java versions

| Application runs on | Local | Remote |
|---|---|---|
| Java 8 JDK | Yes | Yes |
| Java 8 JRE (no `tools.jar`) | Yes | No: "This application runs on a Java 8 JRE, which can't load SQL Blame while it runs." Run it on a JDK or start it from the IDE |
| Java 9 to 20 | Yes | Yes |
| Java 21 and later | Yes, see JEP 451 below | Yes, see JEP 451 below |

Tested: local attach on Java 8, 17, 21 and 25, and remote attach on Eclipse Temurin 8 (JDK and JRE), 17, 21 and 25
containers. Both ways need a HotSpot-based JVM (OpenJDK builds, Oracle JDK, Temurin, Corretto, JBR, Zulu and
similar). OpenJ9 and other JVMs are not supported unless they offer the same attach mechanisms.

### Java 21 and later (JEP 451)

Since Java 21 a JVM prints a warning on its standard error when an agent is loaded while it runs, starting with
"WARNING: A Java agent has been loaded dynamically". It is only a warning and the application continues.

A JVM started with `-XX:-EnableDynamicAgentLoading` refuses the load, and a future JDK may refuse it by default. SQL
Blame then says "This application doesn't allow loading agents while it runs. Start it with
-XX:+EnableDynamicAgentLoading." and offers **Copy Flag**. Adding `-XX:+EnableDynamicAgentLoading` also removes the
warning.

## Messages

| Message | What to do |
|---|---|
| This application doesn't allow loading agents while it runs. | Start it with `-XX:+EnableDynamicAgentLoading` (**Copy Flag**). |
| This application runs as another user. | Local attach works only within your own OS user. Start the application as your user, or give it a debug port and attach remotely. |
| Can't reach dev-box.local:5005. | Check that the application runs with a debug port, that `address=` accepts outside connections (`*:5005`) and that the port is published or forwarded (**Edit Target**). |
| A debugger is already connected to this port. | JDWP takes one debugger at a time. Stop the IntelliJ debug session (or other debugger) on that port and attach again. |
| The application stayed idle for 30 s. | Use the application once, for example send a request, and click **Try Again**. |
| This application runs on a Java 8 JRE, which can't load SQL Blame while it runs. | Run it on a JDK, or start it from the IDE so the agent comes with the start. |
| SQL Blame can't attach to this JVM. | The JVM offers no attach mechanism SQL Blame knows (see Java versions). |
| SQL Blame needs a license or an active trial to capture the SQL of an application. | See [Do I need a license?](faq.md#do-i-need-a-license). |
| SQL Blame couldn't attach to this application. | Something unexpected failed. **Details** opens `idea.log`. The application is not affected, so try again. |

The message also stays on the run's row in the Processes tab until **Clear All Ended**.

## Explain Plan on attached runs

Plans work as described in [explain.md](explain.md): the agent captures the plan of each query's first execution
and explains again on request while the application runs. After the application ended, the Explain Worker connects
with the login the application used. For a remote application that login often names a host the IDE cannot reach,
such as a database container called `db` or the remote machine's own `localhost`. The plan tab then says that SQL
Blame cannot reach the database of this remote run from the IDE and offers **Connect to Database...**, where you
enter an address the IDE can reach. `localhost` in a remote application's JDBC URL is read as the remote machine
when the debug host is another machine. Through a published port or a port forward the IDE cannot tell, and leaves
the URL as the application wrote it.

## Not covered

- A remote JVM without a debug port. A JMX port alone cannot bring an agent in.
- Connections the application opened before the attach and still holds (see above).
- JVMs that are not HotSpot-based, unless they offer the same attach mechanisms.
- Remote applications on a Java 8 JRE.
- Unloading the agent: Detach stops capturing, the classes stay until the JVM ends.

## Safety and privacy

Attaching never stops the application. The agent follows the same rule as in every run: any internal error turns it
off and the application continues unchanged.

A debug port lets anyone who reaches it run code in the application. SQL Blame does not open one, it only uses one
you opened. Keep debug ports on development and test environments and off public networks.

Nothing goes to any server of ours. The IDE sends the agent jar and its options to the application, and the
application sends the captured events back to the IDE, protected by the run's token. For a remote application these
events (SQL, parameter values, call paths) travel over your network without encryption, like the debugger connection
itself. Turn on **Mask parameter values** in Settings | Tools | SQL Blame if the values must not cross it.

## Settings

Settings | Tools | SQL Blame | Attach has three entries. **Ask before a remote attach** brings back the confirmation
after you chose "Don't ask again". **Pass this project's packages to an attached application** is on by default.
When it is off, the agent reads the packages from the application's main class or jar, and packages typed on the SQL
Blame
page always win. **Saved Remote Targets** lists the targets of the Remote tab to rename or remove.
