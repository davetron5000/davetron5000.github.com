--- 
wordpress_id: 39
title: Using ThreadLocal and Servlet Filters to cleanly access JPA an EntityManager
wordpress_url: http://www.naildrivin5.com/daveblog5000/?p=39
layout: post
---
My current project is slowly moving from JDBC-based database interaction to JPA-based.  Following good sense, I'm trying to change things as little as possible.  One of those things is that we are deploying under Tomcat and not under a full-blown J2EE container.  This means that EJB3 is out.  After my <a href="http://www.naildrivin5.com/daveblog5000/?p=37">post regarding this configuration</a>, I quickly realized that my code started to get littered with:

```java
EntityManager em = null;
try
{
  em = EntityManagerUtil.getEntityManager();
  // do stuff with entity manager
}
finally
{
  try {
    if (em != null) em.close();
  } catch (Throwable t) {
    logger.error("While closing an EntityManager",t);
  }
}
```

Pretty ugly, and seriously annoying to have to add <b>13 lines of code</b> to any method that needs to interact with the database.  The Hibernate docs suggest using <tt>ThreadLocal</tt> variables to provide access to the EntityManager throughout the life of a request (which wouldn't really work for a Swing app, but since this is servlet-based, it should work fine).    The <tt>ThreadLocal</tt> javadocs contain possibly the most annoying example ever, and I didn't follow how to use it.

Anyway, I finally got around to it, and also solved the close problem as well, by using a Servlet Filter.  I guess this type of thing would normally be solvable by Spring or Guice, but I didn't want to drag all of that into the application to refactor this one thing; I would've easily spent the rest of the day dealing with XML configuration and deployment.

The solution was quite simple:

```java 
/** Provides access to the entity manager.  */
public class EntityManagerUtil 
{
    public static final ThreadLocal<EntityManager> 
        ENTITY_MANAGERS = new ThreadLocal<EntityManager>();

    /** Returns a fresh EntityManager */
    public static EntityManager getEntityManager()
    {
        return ENTITY_MANAGERS.get();
    }
}
```

```java
public class EntityManagerFilter implements Filter
{
    private Logger itsLogger = Logger.getLogger(getClass().getName());
    private static EntityManagerFactory theEntityManagerFactory = null;

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException
    {
        EntityManager em = null;
        try
        {
            em = theEntityManagerFactory.createEntityManager();
            EntityManagerUtil.ENTITY_MANAGERS.set(em);
            chain.doFilter(request,response);
            EntityManagerUtil.ENTITY_MANAGERS.remove();
        }
        finally
        {
            try 
            { 
                if (em != null) 
                    em.close(); 
            }
            catch (Throwable t) { 
                itsLogger.error("While closing an EntityManager",t); 
            }
        }
    }
    public void init(FilterConfig config)
    {
        destroy();
        theEntityManagerFactory = 
          Persistence.createEntityManagerFactory("gliffy");
    }
    public void destroy()
    {
        if (theEntityManagerFactory != null)
            theEntityManagerFactory.close();
    }
}
```

So, when the web app gets deployed, the entity manager factory is created (and closed when the web app is removed).  Each thread that calls <tt>EntityManagerUtil</tt> to get an EntityManager gets a fresh one that persists for the duration of the request.  When the request is completed, the entity manager is closed automatically.
