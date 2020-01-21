--- 
title: The only four types of classes in your OO system
layout: post
ad:
  title: "SOLID Doesn't Help"
  subtitle: "Learn how much worse it makes things"
  link: "http://bit.ly/not-solid"
  image: "/images/not-solid-cover.png"
  cta: "Buy Now $5.99"
---
Object-Oriented design is hard, especially in a large application.  It's not always clear where logic should go, and there's often no "right place" to  put a piece of code.  I've found that there are four distinct types of classes that, if you stick to them, can make your code a lot more understandable, and can provide clear direction as to the age-old question of "where does this code go?"

## 0. Background
The J2EE way is to have model objects be stupid structs, and have all business logic in a service layer (this is actually very close to a classic "functional programming" way of doing things; ironic that many Java devs eschew FP).  Spring lets you do whatever you want, but more or less follows this pattern.  
While the "Rails Way" is to put business logic on the model objects, I think the "service layer" concept <a href="http://www.engineyard.com/blog/2010/let-them-code-cake/">is eventually going to be common practice</a>.
So, I've found that I very rarely make a pure "by-the-book according-to-Hoyle" OO-compliant class; I've settled on four patterns that seem to cover pretty much everything.  I've also noticed that when these patterns get mixed together, you get trouble.

## 1. The Record

The <i>record</i> is a dumb struct that you usually need to appease your object-relational mapper.  You may need them elsewhere to just name and type some set of data that you either can't model as a tuple because of your language, or don't want to model as a tuple because of some complexity.  A record typically has methods that merely expose it's contents and often need to be mutable for the reasons stated.  You might have derived fields that are convieniences and not based on your core business logic.  An easy example is a person.  They have a name and a birthdate, and you might derive their age from that:

```java
public class Person {
  private String name;
  private Date birthdate;
 
  public Person(String name, Date birthdate) {
    this.name = name;
    this.birthdate = birthdate;
  }
 
  public String getName() { return this.name }
  public void setName(String name) { this.name = name; }
 
  public int getBirthdate() { return this.birthdate; }
  public void setBirthdate(Date birthdate) { 
    this.birthdate = birthdate;
  }
 
  public int getAge() {
    // I know this is slightly buggy :)
    return (
      new Date().getTime() 
      - getBirthdate().getTime()
      ) / (1000 * 60 * 60 * 24 * 365);
  }
  // Maybe some toString, equals, etc. type stuff as well
}
```

## 2. The Immutable Object

This is the closest to a pure "object-oriented" design.  Classes of this type are immutable and should hold data you will use a lot in your system.  They may also probably have some business-logic attached as methods; this business logic should be entirely focused on the object and its contents.  Typical methods will give you more complex information about the data the object contains, or will vend new objects of the same type, based on the method and parameters called.  


This is the most clear distinction (in my mind) between functional programming and object-oriented programming.  In an FP world, the data being operated on would be loosely defined (if at all) and you'd have functions that transform it.  In an OO world, your object's data is clearly defined (by the class fields/accessors), and the operations available are the methods of the class.  When you require that the objects of the class be immutable, you have a very nice encapsulated package of data and operations.  This, to me, seems a lot easier to deal with than a "module" of functions and some tuples (or lists of tuples) that the functions operate on.  Scala makes it very easy to create classes like this.  It's probably one of the few languages that does so (Java certainly is no help, but it <b>can</b> be done).

```java
public class Appointment {
  private final Date date;
  private final String description;
  private final Collection<Person> attendees;
 
  public Appointment(
      Date date, 
      String description, 
      Collection<Person> attendees) {
    // normally, you would validate the inputs
    // for sanity, e.g. Validate.notNull(date)
 
    // Since Date is mutable, we make a copy
    this.date = new Date(date.getTime());
    this.description = description;
    this.attendees = Collections.unmodifiableCollection(attendees);
  }
 
  public Date getDate() {
    // Since Date is mutable, we vend a copy
    return new Date(this.date.getTime());
  }
 
  public String getMessage() {
    return this.message;
  }
 
  public Collection<People> getAttendees() {
    return this.attendees;
  }
 
  public boolean isLate(Date otherDate) {
    return this.date.before(otherDate);
  }
 
  public boolean shouldRemind(Date otherDate) {
    return !isLate() && (otherDate.getTime() 
      - this.date.getTime()) >= (60 * 5 * 1000);
  }
 
  public boolean isAttending(Person p) {
    return this.attendees.contains(p);
  }
 
  public Appointment reschedule(Date newDate) {
    return new Appointment(newDate,getMessage(),getAttendees());
  }
 
  public Appointment notAttending(Person p) {
    if (isAttending(p)) {
      Collection<Person> newGroup = new HashSet<Person>(getAttendees());
      newGroup.remove(p);
      return new Appointment(getDate(),getMessage(),newGroup);
    } 
    else {
      return this;
    }
  }
 
}
```

<div data-ad></div>

The benefits here are huge; immutability allows your codebase to be much more comprehensible, and allows you to use these objects in concurrent situations without worry.  Since they are immutable, their methods are immediate targets for caching if you discover you need to do this to improve performance.

## 3. The Builder
While you can certainly use methods (or create methods) on Immutable Object classes to "build up" the object you want, this is often cumbersome, and results in a lot of object creation for no real reason.  The "builder" can be used to make this a bit simpler.  The Builder is a throwaway class whose sole purpose is to create Immutable Objects.  This obviously creates a very tight coupling between the two classes, but this can be worth it.  This is very preferable to a mutable class and, depending on your operating environment, is preferable to making many intermediate objects you will need to create the Immutable Object.

```java
public class AppointmentBuilder {
  private Date date;
  private String description;
  private List<Person> people;
 
  public AppointmentBuilder setDate(Date date) {
    this.date = date;
    return this;
  }
 
  public AppointmentBuilder setDescription(String description) {
    this.description = description;
    return this;
  }
 
  public AppointmentBuilder addPerson(Person p) {
    this.people.add(p);
    return this;
  }
 
  public Appointment build() {
    return new Appointment(date,description,people);
  }
}
```

## 4. The Service

The analog of The Record, the <i>service</i> has no data and all logic.  EJBs are services; they have no internal state, operating on their arguments and returning a result.  Methods of services can be very functional in nature (operating solely on structs or immutable objects), or they may provide functionality that implements complex business logic not logically part of an immutable object's class.  In a vanilla n-tier application, you use services to get data in and out of your database (you might call these DAOs and you might distinguish different types of services for partitioning, but these are all the same sort of class).

Like records, Services are not OO at all; these are the functions to your C programs structs.  But, there is a good reason for this design; you separate concerns, don't need to worry about concurrency (services have no state), and can even horizontally partition <i>where</i> serivces actually run.

```java
public class Calendaring {
  /** Schedule an appointment */
  public Appointment schedule(
      Date date, 
      String description, 
      String... names) {
    AppointmentBuilder builder = new AppointmentBuilder(date)
      .setDescription(description);
    for (String name: names) {
       Person p = findPersonByName(name);
       if (p != null) {
         builder.addPerson(p);
       }
    }
    return builder.build();
  }
}
```

## Update from 2017

I don't write a lot of Java in 2017, but I do write a lot of Rails, and these patterns have served me well.  Every bit of code
I've written and watched grow over 4+ years that was disciplined, and followed the above patterns, has been easier to understand
and test.  Code that didn't—for example, mixing a record and a service into one class—has been harder to evolve and manage.
