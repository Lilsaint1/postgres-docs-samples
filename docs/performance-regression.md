# Why	Did	My	Application	Get	Slower	After	a	Release?
A	new	release	goes	out.The	 deploy	succeeds.	Health	checks	are	green.	Nothing	has	technically	broken.
Then	the	messages	start:
> *"The	app	feels	a	lot	slower	than	it	did	yesterday."*



No	errors.	No	downtime.	The	database	is accepting connections. The dashboards are mostly	green. So what	actually changed?

A release can quietly hurt	performance	without	breaking	anything	outright:	a	query	that	got	a	little	more	expensive,	a new	API	call	added	to	every	request,	an	index	that	no	longer	matches	how	the	query	is	now	written.	The	application stays	technically	available	while	becoming	practically	unusable,	and	that	gap	is	exactly	where	this	kind	of	incident hides.

This	guide	walks	through	a	systematic	way	to	find	it:

> ___Establish	the	regression	→	Identify	the	affected	layer	→	Collect	evidence	→	Compare	against	the	last release	→	Find	the	root	cause	→	Apply	the	smallest	fix	that	works	→	Verify___

***
> ### Don’t	Start	With	the	Database

The	first	instinct	is	almost	always	“the	database	must	be	slow.”	Sometimes	it	is.	But	the	database	is	just	one	stop	on	a
much	longer	journey:
```

    User	→	Browser	→	Load	Balancer	→ Application	Server
                                       │
                        ┌───────────────┼───────────────┐	
                External API      Cache			PostgreSQL 
```

A	regression	can	appear	at	any	point	on	that	path:	a	chattier	frontend,	an	added	API	call,	a	cache	that	quietly	stopped working,	a	changed	query.	So	the	opening	question	isn’t	“what’s	wrong	with	Postgres?”	It’s:

**Which	part	of	the	request	actually	got	slower?**

That	one	reframe	saves	hours.	It’s	also	the	difference	between	a	junior	debugging	session	and	a	senior	one.

### Confirm	the	Regression	Is	Real
“It	feels	slower”	is	a	useful	signal,	not	a	diagnosis.	Before	investigating	causes,	get	numbers:

|                   |   Before  |   After   |
| ------------      | --------  |   ------- |
| Average	latency |   180 ms  |   620 ms  |
| 95th	percentile  |   350	ms  |   1.8 s   |
| 99th	percentile  |   600 ms  |   4.2 s   |

Now	it’s	measurable.	Next:	when	did	it	start,	relative	to	the	deploy?	A	timeline	makes	the	release	an	obvious	suspect:

```
10:00		Release	deployed
10:08		Latency	starts	climbing
10:15		First	user	reports
10:20		Latency	peaks
It	doesn't	prove	the	release	caused	it,	but	it	gives	you	a	real	hypothesis	instead	of	a	guess.	Correlation	tells you	where	to	look;	evidence	tells	you	what	happened.```
Find the	Slow	Request,	Not	the	Slow	App
```


An	application	almost	never	gets	uniformly	slower.	Usually	one	endpoint	is	dragging	the	average	down:

GET	/api/products	120	ms	GET	/api/orders	4,800	ms	←	here	GET	/api/profile	150	ms	POST	/api/payment	400	ms

```
That	turns	*"the	app	is	slow"*	into "/api/orders`	got	roughly	20x	slower	after	release	4.8." That's	a	problem	an engineer	can	actually	pick	up	and	run	with. 
###	Break	the	Request	Into	Pieces
A	single	number,	4.8	seconds,	doesn't	tell	you	where	the	time	went.	Instrumenting	the	request	does:
```

Total	request:	4,800	ms	Authentication	20	ms	Application	logic	150	ms	External	API	100	ms	Database	4,400	ms	←	the database	is	the	story	here	Serialization	80	ms

But	compare	that	to	a	different	breakdown	of	the	*same	total*:
Total	request:	4,800	ms	Authentication	20	ms	Application	logic	150	ms	External	API	4,400	ms ←	now	it	isn’t	Database 120 ms
```

Same	headline	number,	completely	different	root	cause.	This	is	the	single	most	useful	habit	in	this	whole	process: measure	the	*whole*	request	before	assuming	which	piece	is	guilty.

###	When	the	Database	Is	Innocent
Sometimes	the	query	is	fine,	and	the	application	is	just	asking	for	more	of	them.	The	classic	version	of	this	is	the	**N+1	query	problem**:	fetch	100	orders,	then	fetch	each	order's	customer	with	a	separate	query.	That's	101	round	trips where	a	handful	would	do.
Each	individual	query	might	run	in	20–80ms	and	look	completely	healthy	in	isolation.	It's	only	when	you	count	*how	many	times*	the	app	is	asking,	not	just	*how	fast*	each	ask	returns,	that	the	real	cost	shows	up.	The	fix	(usually	a	join,	or batching	the	lookups)	matters	less	than	the	diagnostic	habit	behind	it:	**count	the	work,	not	just	the	time.**
The	same	shape	shows	up	elsewhere	too:	a	release	that	doesn't	change	the	SQL	at	all,	but	changes	*how	often*	it	runs:
```

Before:	100	requests	×	1	query	=	100	queries	After:	100	requests	×	20	queries	=	2,000	queries

```
Perfectly	optimized	SQL,	twenty	times	the	load.	Query	*cost*	and	query	*frequency*	are	two	different	questions,	and	a	regression	can	hide	in	either	one.
*(If	the	database	genuinely	is	the	bottleneck,	the	usual	next	steps,	reading	the	execution	plan,	checking	whether	it's executing	or	stuck	waiting	on	a	lock,	follow	the	same	evidence-first	approach	as	any	query-performance	investigation.)*

###	The	Bottleneck	Is	Often	Outside	the	Database	Entirely

Three	causes	worth	knowing	by	name,	because	they're	easy	to	miss	if	you're	only	looking	at	Postgres:
**A	new	external	dependency.** A	release	adds	a	fraud-check	API.	Your	queries	haven't	changed	at	all,	but	now	every request waits	3	extra	seconds	on	someone	else's	service:
```

Application:	120	ms	PostgreSQL:	80	ms	Fraud	API:	3,200	ms	←	the	actual	story

```
**A	retry	policy	that	multiplies	the	problem.**	A	dependency	that	used	to	fail	fast	now	retries	three	times	at	2	seconds	each.	That	turns	one	slow	call	into	six	seconds	of	waiting,	and	multiplies	downstream	traffic	in	the	process:```

Slow	dependency	→	timeout	→	retry	→	more	traffic	→	more	load	→	slower	dependency	→	more	timeouts

```
That's	not	a	performance	regression	anymore.	Left	unchecked,	it's	a	feedback	loop	that	becomes	an	outage.
```


**A	cache	that	quietly	stopped	working.**	A	changed	cache	key,	a	changed	expiration	policy,	a	changed	serialization format,	and	hit	rate	drops	from	95%	to	40%	overnight.	The	database	didn't	get	less	efficient.	It	just	stopped	being protected.

###	Compare	the	Same	Request,	Two	Versions
The	cleanest	way	to	close	out	an	investigation	like	this	is	a	direct,	controlled	comparison:

v4.7:	Request	latency	180	ms	(DB:	40	ms	·	External	API:	20	ms	·	App:	120	ms)	v4.8:	Request	latency	2,800	ms	(DB:	45
ms	·	External	API:	2,600	ms	·	App:	155	ms)

The	database	barely	moved.	The	evidence	points	somewhere	else	entirely,	in	this	case	straight	at	the	new	external
dependency,	not	at	PostgreSQL.	That’s	the	payoff	of	measuring	the	whole	path	instead	of	guessing	from	the	symptom: the	fix	goes	to	the	actual	cause	on	the	first	try,	instead	of	an	index	that	was	never	going	to	help.

> ### The	Actual	Takeaway

None of	this is about memorizing a list of	possible	causes:	code, migrations, config, connection pools, caching, external services, retries. It’s about	resisting the pull toward the nearest familiar suspect	(almost	always	“the	database”) and instead following one question all the way down.

**Which	part	of	the	request	actually	got	slower,	and	what	evidence	says	so?**
Everything	else,	the	timeline,	the	request	breakdown,	the	version	comparison,	exists	to	answer	that	one	question honestly,	instead	of	guessing	and	hoping	the	fix	sticks