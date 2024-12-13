# Flink DataStream Socket Source and Print Sink

- [Intro](#intro)
- [Setup](#setup)
- [Walkthrough for flink datastream api, socket as source and print as sink](#walkthrough-for-flink-datastream-api-socket-as-source-and-print-as-sink)
- [Tests](#tests)
  - [Simple test](#simple-test)
  - [Random string](#random-string)
  - [Shakespeare](#shakespeare)
  - [Long string](#long-string)

# Intro

In this tutorial, we will walkthrough how to use flink datastream api to read data from socket and print the result.

# Setup

First, let's set up a new flink project. We can use the official flink quickstart script to create a new project.

```bash
bash -c "$(curl https://flink.apache.org/q/gradle-quickstart.sh)" -- 1.15.0 _2.12
```

# Walkthrough for flink datastream api, socket as source and print as sink

Let's see how to initialize `StreamExecutionEnvironment` and setup print sink.

We start by creating a `StreamExecutionEnvironment`, which is the main entry point for all Flink applications. We then create a `DataStream` by adding a source function that reads from a socket and a sink function that prints its input to the console.

```java
public class DataStreamJob {

    public static void windowWordCount() throws Exception {
        // StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(value -> value.f0)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static void main(String[] args) throws Exception {
        // Sets up the execution environment, which is the main entry point
        // to building Flink applications.
        // final StreamExecutionEnvironment env =
        // StreamExecutionEnvironment.getExecutionEnvironment();

        windowWordCount();
    }
}
```

The `windowWordCount` method first creates a datastream from a socket, then splits the words from the text into a tuple of (word, 1) and groups by the word. It then creates a window of 5 seconds and sums the counts of each word in the window. The result is then printed to the console.

Run local socket using nc:

```bash
nc -lk 9999
```

Build flink job:

```bash
gradle clean installShadowDist
```

Submit flink job:

```bash
FLINK_HOME=~/flink/flink-1.15.1
$FLINK_HOME/bin/flink run -c org.myorg.quickstart.DataStreamJob build/install/quickstart-shadow/lib/quickstart-0.1-SNAPSHOT-all.jar
```

The output will be like:

```
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by org.apache.flink.api.java.ClosureCleaner (file:/home/ec2-user/flink/flink-1.15.1/lib/flink-dist-1.15.1.jar) to field java.lang.String.value
WARNING: Please consider reporting this to the maintainers of org.apache.flink.api.java.ClosureCleaner
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
Job has been submitted with JobID 674851e1ff9da68eb742f93d6d874ca6
```

# Tests

Let's see how to test the job.

## Simple test

We can test it by using `nc` command.

Type in terminal running `nc` command:

```bash
123 555
hello world
hello world
hi jon
hi non
hello 123
```

Check log in `$FLINK_HOME/log` directory, file `flink-ec2-user-taskexecutor-1-ip-172-24-145-118.cn-northwest-1.compute.internal.out`:

```bash
==> flink-ec2-user-taskexecutor-1-ip-172-24-145-118.cn-northwest-1.compute.internal.out <==
(world,1)
(hello,1)
(555,1)
(hello,1)
(world,1)
(hi,2)
(non,1)
(jon,1)
(hello,1)
(123,1)
```

## Random string

`nc`:

```
a b c a b c a b c hello non hello k
```

flink log:

```
==> flink-ec2-user-taskexecutor-1-ip-172-24-145-118.cn-northwest-1.compute.internal.out <==
(a,3)
(k,1)
(non,1)
(hello,2)
(c,3)
(b,3)
```

## Shakespeare

input

```
To be, or not to be, that is the question
```

log

```
==> flink-ec2-user-taskexecutor-1-ip-172-24-145-118.cn-northwest-1.compute.internal.out <==
(To,1)
(question,1)
(the,1)
(is,1)
(that,1)
(to,1)
(not,1)
(or,1)
(be,,2)
```

## Long string

input

```
To be, or not to be, that is the question:
Whether 'tis nobler in the mind to suffer
The slings and arrows of outrageous fortune,
Or to take arms against a sea of troubles
And by opposing end them. To die—to sleep,
No more; and by a sleep to say we end
The heart-ache and the thousand natural shocks
That flesh is heir to: 'tis a consummation
Devoutly to be wish'd. To die, to sleep;
To sleep, perchance to dream—ay, there's the rub:
For in that sleep of death what dreams may come,
When we have shuffled off this mortal coil,
Must give us pause—there's the respect
That makes calamity of so long life.
For who would bear the whips and scorns of time,
Th'oppressor's wrong, the proud man's contumely,
The pangs of dispriz'd love, the law's delay,
The insolence of office, and the spurns
That patient merit of th'unworthy takes,
When he himself might his quietus make
With a bare bodkin? Who would fardels bear,
To grunt and sweat under a weary life,
But that the dread of something after death,
The undiscovere'd country, from whose bourn
No traveller returns, puzzles the will,
And makes us rather bear those ills we have
Than fly to others that we know not of?
Thus conscience doth make cowards of us all,
And thus the native hue of resolution
Is sicklied o'er with the pale cast of thought,
And enterprises of great pith and moment
With this regard their currents turn awry
And lose the name of action
```

log

```
==> flink-ec2-user-taskexecutor-1-ip-172-24-145-118.cn-northwest-1.compute.internal.out <==
(,2)
(action,1)
(name,1)
(lose,1)
(awry,1)
(turn,1)
(currents,1)
(their,1)
(regard,1)
(moment,1)
(pith,1)
(great,1)
(enterprises,1)
(thought,,1)
(cast,1)
(pale,1)
(with,1)
(o'er,1)
(sicklied,1)
(Is,1)
(resolution,1)
(hue,1)
(native,1)
(thus,1)
(all,,1)
(cowards,1)
(doth,1)
(conscience,1)
(Thus,1)
(of?,1)
(know,1)
(others,1)
(fly,1)
(Than,1)
(ills,1)
(those,1)
(rather,1)
(will,,1)
(puzzles,1)
(returns,,1)
(traveller,1)
(bourn,1)
(whose,1)
(from,1)
(country,,1)
(undiscovere'd,1)
(death,,1)
(after,1)
(something,1)
(dread,1)
(But,1)
(life,,1)
(weary,1)
(under,1)
(sweat,1)
(grunt,1)
(bear,,1)
(fardels,1)
(Who,1)
(bodkin?,1)
(bare,1)
(With,2)
(make,2)
(quietus,1)
(his,1)
(might,1)
(himself,1)
(he,1)
(takes,,1)
(th'unworthy,1)
(merit,1)
(patient,1)
(spurns,1)
(office,,1)
(insolence,1)
(delay,,1)
(law's,1)
(love,,1)
(dispriz'd,1)
(pangs,1)
(contumely,,1)
(man's,1)
(proud,1)
(wrong,,1)
(Th'oppressor's,1)
(time,,1)
(scorns,1)
(whips,1)
(bear,2)
(would,2)
(who,1)
(life.,1)
(long,1)
(so,1)
(calamity,1)
(makes,2)
(respect,1)
(pause—there's,1)
(us,3)
(give,1)
(Must,1)
(coil,,1)
(mortal,1)
(this,2)
(off,1)
(shuffled,1)
(have,2)
(When,2)
(come,,1)
(may,1)
(dreams,1)
(what,1)
(death,1)
(For,2)
(rub:,1)
(there's,1)
(dream—ay,,1)
(perchance,1)
(sleep;,1)
(die,,1)
(wish'd.,1)
(be,1)
(Devoutly,1)
(consummation,1)
(to:,1)
(heir,1)
(flesh,1)
(That,3)
(shocks,1)
(natural,1)
(thousand,1)
(heart-ache,1)
(we,4)
(say,1)
(sleep,2)
(more;,1)
(No,2)
(sleep,,2)
(die—to,1)
(them.,1)
(end,2)
(opposing,1)
(by,2)
(And,5)
(troubles,1)
(sea,1)
(a,5)
(against,1)
(arms,1)
(take,1)
(Or,1)
(fortune,,1)
(outrageous,1)
(of,14)
(arrows,1)
(and,7)
(slings,1)
(The,5)
(suffer,1)
(mind,1)
(in,2)
(nobler,1)
('tis,2)
(Whether,1)
(question:,1)
(the,14)
(is,2)
(that,4)
(to,8)
(not,2)
(or,1)
(be,,2)
(To,5)
```
