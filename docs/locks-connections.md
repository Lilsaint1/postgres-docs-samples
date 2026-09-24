# PostgreSQL	Under	Pressure:	Diagnosing	Locks	and	Connection Problems
----------
When	a	PostgreSQL-backed	application	slows	down,	the	database	is	often	the	first	thing	people	blame.	But	“PostgreSQL is	slow”	is	not	a	diagnosis.

A	database	can	appear	slow	for	reasons	that	have	nothing	to	do	with	a	bad	query:	a	transaction	is	holding	a	lock,	the application	has	exhausted	its	connection	pool,	or	a	session	that	looks	“active”	is	actually	just waiting.	This	guide	uses PostgreSQL’s	own	observability	views	to	separate	those	cases	before	reaching	for	a	fix.

> **Observe	→	Measure	→	Diagnose	→	Change	→	Verify**

Test	destructive	commands	in	a	non-production	environment	first.

> ### Start	With	the	Symptom,	Not	the	Solution**

When	a	team	reports	“the	website	is	slow,	PostgreSQL	is	probably	the	problem,”	it’s	tempting	to	start	creating	indexes or	restarting	the	database	immediately.	Resist	that.	The	first	real	question	is	narrower:


* Are	all	queries	slow,	or	one	endpoint?
* Did	it	degrade	suddenly,	or	gradually?
* Are	queries	executing,	or	waiting?
* Is	the	database	out	of	connections?

Each	of	these	points	somewhere	different. Guessing	wastes	the	one	advantage	you	have	during	an	incident:	time.



> ### Is	It	Executing,	or	Is	It	Waiting?

==pg_stat_activity==	shows	what	every	session	is	doing	right	now:



```
SELECT	pid,	usename,	datname,	state,
    wait_event_type,	wait_event,
    now()	-	query_start	AS	duration,	query
FROM	pg_stat_activity
WHERE	state	<>	'idle'
ORDER	BY	duration	DESC;
```
This	single	distinction	changes	everything;

|Session  |  wait_event_type  |  What’s	actually happening|
|:------: |    : -------- :   |  :-------------:          |
|   A     |     NULL          |  Actively executing       |
|   B     |     Lock          | Blocked, waiting	on another	transaction |

A	query	that	“looks	slow”	because	it’s	blocked	needs	a	completely	different	fix than	one	that’s	genuinely	CPU-heavy.
Conflating	the	two	is	the	most	common	troubleshooting	mistake	there	is.


> ### Finding	the	Blocking Session  

PostgreSQL’s pg_locks	view,	joined	against pg_stat_activity, answers	the	question	that	actually	matters:	**who	is waiting,	and	who	is	preventing	it	from	proceeding?**


```
SELECT	blocked.pid	AS	blocked_pid,	blocked.query	AS	blocked_query,
    blocking.pid	AS	blocking_pid,	blocking.query	AS	blocking_query
FROM	pg_stat_activity	AS	blocked
JOIN	pg_locks	AS	blocked_locks	ON	blocked.pid	=	blocked_locks.pid
JOIN	pg_locks	AS	blocking_locks
    ON	blocked_locks.locktype	=	blocking_locks.locktype
    AND	blocked_locks.relation	IS	NOT	DISTINCT	FROM	blocking_locks.relation
    AND	blocked_locks.pid	<>	blocking_locks.pid
JOIN	pg_stat_activity	AS	blocking	ON	blocking.pid	=	blocking_locks.pid
WHERE	NOT	blocked_locks.granted	AND	blocking_locks.granted;
```
Once	you’ve	found	the	blocking	session, **don’t terminate	it	as	a	reflex**. It might	be	a	legitimate	migration,	a	long report,	or	a	transaction	someone	genuinely	needs	to	finish.	Before	acting,	know	who	owns	it,	what	it’s	doing,	and	what breaks	if	you	kill	it   pg_cancel_backend()	and	pg_terminate_backend()	have	different	effects,	so	use	them	deliberately,	not defensively.

Long-running	*transactions*	deserve	the	same	scrutiny	as	long-running	queries.	A	transaction	left	open	while	the application	does	unrelated	work	can	hold locks	far	longer	than	the	query	itself	would	suggest:


```
SELECT	pid,	usename,	xact_start,	now()	-	xact_start	AS	transaction_duration,	state,	query
FROM	pg_stat_activity
WHERE	xact_start	IS	NOT	NULL
ORDER	BY	xact_start;
```
***
> ### Connection	Pressure	Is	a	Different	Problem


Sometimes	the	symptom	isn’t	a	slow	query	at	all.	It’s	the	application	failing	to	get	a	connection	in	the	first	place.

```
SHOW	max_connections;

SELECT	count(*)	AS	connections,	datname
FROM	pg_stat_activity
GROUP	BY	datname
ORDER	BY	connections	DESC;
```

Raw	connection	count	isn’t	the	real	signal.	The	state	breakdown	is.	An	ordinary right	now;	that’s	normal. ==idle	in	transaction==	is	the	one	worth investigating:	it	means	a	transaction	is	open,	not	doing anything,	and	possibly	still	holding	resources.

```
SELECT	pid,	usename,	xact_start,	state,	query
FROM	pg_stat_activity
WHERE	state	=	'idle	in	transaction'
ORDER	BY	xact_start;
```

Connection	pools	make	this	worse	when	misconfigured,	not	better.	The	math	is	easy	to	overlook:

```
10	application	instances	×	100	connections	per	instance	=	1,000	potential	connections
```


If	PostgreSQL	isn’t	sized	for	that,	the	application	can	exhaust	connections	long	before	the	database	itself	is	under	real
load.	The	instinct	to	raise max_connections	treats	the	symptom,	not	the	cause.	The	better	question	is	why	does	the
application	need	this	many	simultaneous	connections,	not	how	do	I	let	PostgreSQL	accept	more.


> ### A	Decision	Tree,	Not	a	Command	List

**Are	queries	slow?**	→	Check	the	execution	plan,	row	estimates,	and	scan	strategy.	**Are	queries	waiting?**	→	Check wait_event_type	in	pg_stat_activity ,	then pg_locks	if	it’s a	lock. **Are	connections	exhausted? →**	Compare	active	usage
max_connections ,	then	look	at	pool	configuration	and idle	in	transaction	sessions.	**Did	something	change recently?**	→	Deployments,	schema	changes,	and	traffic	shifts	are	the	most	commonly	overlooked	cause	of	all.


> ### What	Makes	This	Worth	Documenting	Well

A	troubleshooting	doc	that	just	lists commands	isn’t	that	useful.	The	version	worth	writing	answers,	for	every	step:	what does	this	show,	why	does	it	matter,	and	what	should	I	do	with	the	result?

Weak	documentation	says:	“Run	this	command	to	see	active	queries.”
    
Better	documentation	says: “Use pg_stat_activity	to	tell whether	the	database	is	executing	or	waiting. A	query	blocked on	a	lock	needs	a	different fix	than	one	consuming	CPU,	and	that distinction	is	usually	invisible	until	you	look at wait_event_type.”

That	gap,	between	a	command	and	the	decision	it	should	lead	to,	is	the	actual	job.


> ### Checklist:	Locks	and	Connections

**Lock	contention	-**	[	]	Identify	the	waiting	session	and	the	one	blocking	it	-	[	]	Determine	how	long	the	blocking
transaction	has	been	open	-	[	]	Confirm	it’s	expected	before	cancelling	or	terminating	anything	-	[	]	Verify	blocked	work actually	resumes	afterward
**Connection	pressure	-**	[	]	Compare	active	connections	against	
max_connections	-	[	]	Look	specifically	for	idle	in transaction	-	[	]	Review	the	application’s	pool	size	against	instance	count	-	[	]	Resist	raising	the	connection	limit	before
understanding	the	cause


Most	PostgreSQL	incidents	aren’t	solved	by	a	clever	command.	They’re	solved	by	asking	the	right	question	first:	is	this
executing,	or	is	it	waiting?	*Everything	else	follows	from	the	answer.*