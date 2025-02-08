---
layout: post
title: "Complexity Management in Software Development (Typescript)"
date: 2024-02-11 12:00:00 +0
categories: blog
---

Working on a long time project the importance of decisions that minimize tomorrow’s work is greater every day. Projects tend to grow in size, capabilities, and obviously complexity.

Let’s quickly see what the complexity is, how to deal with it, and most importantly apply something practical to the code. Code examples provided in this article are in TypeScript, either NodeJS or React.

<!--more-->

## What is complexity

According to Wikipedia, [Programming Complexity](https://en.wikipedia.org/wiki/Programming_complexity) is:

> a term that includes software properties that affect internal interactions. As the number of entities increases, the number of interactions between them increases exponentially, making it impossible to know and understand them all.

Complexity itself is a fascinating topic, due to how subjective it is. And in programming it ranges from simple lines of code to intricate systems with many interacting components. It stretches from the lowest level of implementation in the code to the highest abstractions. It can be measured mathematically, though today we will talk about it as an abstract concept.

## Dealing with Complexity

Let’s be clear from the beginning — _there’s no way to eliminate complexity_, nor there’s a way to stop it from growing.

You can only _manage complexity_ or direct it elsewhere. The concept is surprisingly similar to the golden rule of mechanics, which can roughly be stated as:

> whatever you lose in power you gain in displacement

Both concepts emphasize balance. In mechanics, power and displacement offset each other; in programming, simplifying one part often complicates another. The key is to find an equilibrium that maximizes efficiency.

Imagine working on a big monolith backend. It’s massive, contains lots of code, uses many different technologies, responsible for doing a variety of things. Now imagine how it’s to maintain it:

- it takes substantial amount of time to load the code into IDE;
- it’s version control system is flooded with changes that are not relevant to your task;
- it boots up dependencies of the parts of the code that you may never reach;
- and many other familiar complications of working with one big thing.

Then imagine such a monolith miraculously being split into a set of micro-services. Reflect on how to work with it:

- IDE loads quicker because you’re working with only a certain service at a time;
- contributions of your colleagues go unnoticed by you, because you’re working on separate services;
- code is not bloated just because some part of the application requested a new dependency.

For a brief moment, you may experience a feeling that you have solved complexity, “divide and conquer” as they say. You’ve addressed a single big project as a composition of smaller parts that are easier to work with, and whose business logic now can be kept in mind due to the granularity of your services. But did you really solve it?

- One service may be up, while another is down. This was never a problem with your monolith;
- A contribution to a reusable part has to be performed at least in three places: source code, and both places where it’s used. Meaning instead of the usual single PR you make three;
- A regularly occurring dependency update now has to be performed in multiple places, instead of one.

So now you come to the realization that you did not solve the complexity, but directed it elsewhere. At this point, you may also realize when thinking in terms of complexity it’s important to see _entities and interactions_ between them.

Essentially _every new entity and every new interaction increases complexity_. With every addition, there are now more nodes and interactions to maintain:

- every new functionality requires testing;
- every new dependency requires updates;
- every new import adds another reference.

Therefore we can confidently say that complexity increases as the codebase gets bigger. Important to know this is a natural process, you should not be afraid of it, should not fight it, and instead, you should be mindful of such rules whenever you make a decision. A change today affects your flexibility tomorrow. In order to thrive in the long term, you should always ask yourself questions like:

- Will this change be maintainable after a month or a year?
- Will this change require an explanation to a new developer?
- Does it affect the total amount of components or interactions?

In addition to decomposition, there are also other ways to direct complexity, such as:

- Writing tests. This, of course, will require additional code maintenance, but at least the complexity of maintaining the logic of your app is covered and explained by the test
- Higher level tooling. Instead of writing your own helpers and utils, you may want to use an external package. Now you don’t need to maintain a bunch of code, write tests or documentation for it, but now you need to manage yet another dependency
- Automated processes. The complexity of holding a certain knowledge can be delegated to an automated process, for example, code formatters and linters. On PR reviews you don’t need anymore to spend time explaining consistency or potential issues of using certain specifics of the language, but now you need to maintain the configuration and dependencies of your tooling

## Practical solutions

Luckily, there’s a whole set of ways code can be written to manage complexity. To be fair, these are not entirely solutions, but rather approaches that will help to keep your codebase maintainable and convenient to develop further.

### Inversion of control

The good old [Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control) principle is one of the most helpful ways to split responsibilities between participants and therefore delegate a certain amount of complexity to corresponding parties.

Imagine a processor that publishes content to another platform. When publishing is complete it will do a set of actions (send notification, add a record to activity log):

```typescript
type Content = Record<string, string>;
type Originator = "user" | "automation";

const processContent = (content: Content, originator: Originator) => {
  const parsedContent = parseContent(content);
  const publishContent = publishContent(parsedContent);

  if (originator === "user") {
    activityLog.addRecord("User has published content");
    notification.send("Your content is published");
  }

  if (originator === "automation") {
    activityLog.addRecord("Automation has published content");
    notification.send("Your content is published automatically!");
  }

  somethingElse();
};
```

The logic here can be split into 3 main parts: general, user-specific, and automation-specific functionality. But imagine how big this processor would be if there were 10 different originators?

The inversion of control principle basically tells us that the originator-specific logic is none of the processor’s concern — its job is simply to parse and publish. So what we can do is inverse the control of extra logic to a higher level of abstraction:

```typescript
type Content = Record<string, string>;

// General functionality
const processContent = (content: Content, onPublish: () => void) => {
  const parsedContent = parseContent(content);
  const publishContent = publishContent(parsedContent);

  onPublish();

  somethingElse();
};

// User specific functionality
const processUserContent = (content: Content) => {
  const onPublish = () => {
    activityLog.addRecord("User has published content");
    notification.send("Your content is published");
  };

  processContent(content, onPublish);
};

// Automation specific functionality
const processAutomationContent = (content: Content) => {
  const onPublish = () => {
    activityLog.addRecord("Automation has published content");
    notification.send("Your content is published automatically!");
  };

  processContent(content, onPublish);
};
```

This way no matter how many originators we add — their code doesn’t interfere.

### Premature generalization

Focusing on the changes of tomorrow we tend to create more flexibility than actually needed. Flexibility usually means the distribution of logic between moving parts, but this increases complexity without bringing value today. It’s a very thin line between how flexible and how specific your code should be and always requires good consideration.

The problem is that if you convert a single entity of code to a set of interacting parts, you create a sub-system that requires maintenance.

Do not over abstract a function beforehand:

```typescript
// Prematurely generalized and overly abstracted approach
const processData = (data: any): string => {
  return JSON.stringify(data);
};

const valdiateData = (data: any): boolean => {
  return !!data;
};

function processContent<TData>(
  data: TData,
  process: (d: TData) => TData,
  validate: (d: TData) => boolean,
): TData {
  if (!validate(data)) {
    throw new Error("Invalid data");
  }

  return process(data);
}

const data = { hello: "world" };
const result = processContent(data, processData, validateData);

// A better approach with more specific code
// that makes sense when there are no other usages
function processContent(data: any) {
  if (!data) {
    throw new Error("Invalid data");
  }

  return JSON.stringify(data);
}
```

The same applies to cases when TypeScript generics are used in places with not enough variations in the usage:

```typescript
// Bad - unnecessary generalized
class DataManager<
  TData,
  TProcessor = DataProcessor,
  TValidator = DataProcessor,
> {
  constructor(
    public data: TData,
    public processor: TProcessor,
    public validator: TValidator,
  ) {}
}

// Good - generalization is applied only on the parts
// that are different from usage to usage
class DataManager<TData> {
  constructor(
    public data: TData,
    public processor: DataProcessor,
    public validator: DataValidator,
  ) {}
}
```

And it’s also applicable for regular type definitions:

```typescript
// Bad - overly generalized
type ComplexData<TData, TKey extends string | number, TLabel = string> = {
  data: TData;
  key: TKey;
  label?: TLabel;
};

// Good - generalization is applied only on the parts
// that are different from usage to usage
type ComplexData<TData> = {
  data: TData;
  key: string | number;
  label?: string;
};
```

### Performance vs maintainability

Same tradeoffs you have to make when you choose before more performant or easily maintainable code. There are different things such as mutability, low lever operators, and constructs to choose from, but beware that it always comes at a cost — either in maintenance or performance.

```typescript
// Optimzied
function reverseString(str: string): string {
  let result = "";
  for (let i = str.length - 1; i >= 0; i--) {
    result += str[i];
  }
  return result;
}

console.log(reverseString("hello")); // Output: "olleh"

// Readable
function reverseString(str: string): string {
  return str.split("").reverse().join("");
}

console.log(reverseString("hello")); // Output: "olleh"
```

In the examples above optimized version is likely quicker for large strings. It avoids creating an array and then reversing it, which can be more memory-intensive and slower for large strings. Instead, it builds the reversed string directly in a single loop. That’s exactly your job to decide whether sacrificing readability and maintainability is worth it.

### Less syntax sugar

Nowadays Typescript and other tools provide so much syntax sugar that is meant to be a simplification of certain code, but often people start applying it whenever they can, rendering the majority of the code unreadable and difficult to maintain.

Rules of complexity apply to the simplest forms of signature definitions and their implementations. Please do not overcomplicate code by stuffing as much syntax sugar just to save a couple of lines of code:

```typescript
// Bad - too much syntax sugar
const FancyInput: React.FC<{ value: string; onChange: (e: React.ChangeEvent<HTMLInputElement>) => void; placeholder?: string }> = ({ value, onChange, placeholder }) => (
  <input type="text" value={value} onChange={onChange} placeholder={placeholder} />
);

// Good - structured code that does not rely
// on language versions so much
type Props = {
  value: string;
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  placeholder?: string;
};

const FancyInput: React.FC<Props> = (props) => {
  return <input type="text" value={props.value} onChange={props.onChange} placeholder={props.placeholder} />;
};
```

### Configuration

One of the simple ways to manage complexity is to identify parts and especially values that are most likely to be changed: timeout durations, frequencies of certain operations such as polling, logging depending on the environment, and so on. Instead of scattering this logic around and embedding it into the main functionality we can extract it to a dedicated configuration file.

```typescript
const config = {
 fetchTimeout: 10_000,
 pollInterval: 1_000,
 logLevel: 'info'
}

const logger = {
 info: (message: string, data?: any) => {
  if (config.logLevel === 'production') {
   return;
  }

  console.log(message, data);
 };
}

const veryImportantFunction = () => {
 const resultA = requestA(config.fetchTimeout);
 const resultB = requestB(config.fetchTimeout);

 logger.info('Result: ', { resultA, resultB });

 pollResponse(config.pollInterval);
};
```

Now, if a value changes, we update it in a single file instead of multiple. We save time from browsing the code and looking for all of the requests and polls.

A downside is that we now need to maintain configuration and make sure we don’t lose flexibility. For example, what if now certain requests should abort after 10 seconds, while others only after 5 seconds?

### Design patterns

Generally design patterns, whether it’s creational, structural, or behavioral are there to help you manage the complexity:

- “Strategy” will help you to manage the logic of the application if it has different variations;
- “Facade” will help you to encapsulate and abstract functionalities, making them easier to use;
- “Factory” will help you generate instances and maintain control of the creation.

The main problem is once again — if you introduce a pattern to the system, now you need to follow it, code has to follow certain rules, logic has to be located in certain ways, and this is also a complexity.

Patterns are there to help, but they are not the silver bullet. Balancing is the key.

### Dependencies

This is one of the biggest tradeoffs you can make in your application. Delegating responsibility over certain functionality to a third party is a great way to manage complexity, but no lunch comes for free.

A good example would be a “client” package for making requests like [SWR for React](https://swr.vercel.app/docs/getting-started). In order to handle HTTP requests from your React application you would need to handle multiple other aspects of it: error handling, request progress indication, etc.

```typescript
type FetcherFn = () => Promise<TData>;
const useLoad = <TData, >(fetcherFn: FetcherFn) => {
 useEffect(() => {
  const request = async () => {
   try {
    setLoading(true);
    setError(null);

    const response = await fetcherFn();
    setData(response);
   } catch (err) {
    setError(err);
   } finally {
    setLoading(false);
   }
  };

  request();
 }, [fetcherFn]);

 return state;
};

function Profile () {
  const { data, error, isLoading } = useLoad(fetcher);

  if (error) {
   return <div>failed to load</div>
  }

  if (isLoading) {
   return <div>loading...</div>
  }

  return <div>hello {data.name}!</div>
}
```

Such complexity can be easily delegated to a third-party solution, allowing your code to focus on the actual functionality:

```typescript
import useSWR from 'swr';

function Profile () {
  const { data, error, isLoading } = useSWR('/api/user/123', fetcher)

  if (error) {
   return <div>failed to load</div>
  }

  if (isLoading) {
   return <div>loading...</div>
  }

  return <div>hello {data.name}!</div>
}
```

But as was mentioned, complexity is transformed and never gone. In practice, this means that now you need to keep the library up to date, be aware of compatibility with other technologies in the stack, follow its interfaces instead of yours, be at risk of security issues with the package, etc.

But the more growth we introduce to the system, the more justifiable such delegation becomes, because a third-party solution most likely is more capable, allowing you to perform extra functionality out of the box, i.e. retry, postponed call, logging, and many more.

## Summary

- Complexity doesn’t go away — it can be managed or directed;
- Complexity grows naturally with your application;
- Complexity is balance, and it’s your job to find a golden middle;
- Choices today define complexity and difficulty tomorrow.

### Further resources

There’s still much more to learn about complexity and how to operate it in the growing systems, so these resources may be helpful to explore it further:

- “Programming Complexity” on [Wikipedia](https://en.wikipedia.org/wiki/Programming_complexity)
- “Catalog of Design Patterns” on [Refactoring Guru](https://refactoring.guru/design-patterns/catalog)
- “Lecture Notes on Managing Complexity” from [Stanford](https://web.stanford.edu/~ouster/cgi-bin/cs190-spring15/lecture.php?topic=complexity)

