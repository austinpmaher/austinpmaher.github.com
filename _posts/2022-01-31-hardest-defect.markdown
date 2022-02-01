---
layout: default
title: My OG Multi-thread Defect 
categories:
  - weblog

---

I realized recently that my most challenging defect had just celebrated a landmark birthday and I thought I would celebrate by writing the story down so I could revisit it in my declining years.

## Prelude

As our story opens, the customer has been suffering in production due to excessive memory usage on the solution "web-tier" (application servers that generate web pages). A patch was produced that improved overall memory usage, but load testing to verify the fix uncovered a data integrity problem: data previously entered into a form would sometimes disappear after a save. 
	
The data integrity problem was difficult to reproduce. Initially, it could only be reproduced in production. In November, load testing uncovered a problem with one of the optimizations made in the memory usage patch (incorrectly sharing a non-thread-safe instance of an XML DocumentBuilder across multiple requests). I adjusted that optimization accordingly. 
	
In December the customer put together a load runner script and was able to reliably reproduce the problem in their test environment. If the system was loaded with 100 pseudo-users filling out forms and a real user then repeatedly edited and saved a single form, eventually the error would re-occur. 
	 
Based on some errors I saw in the customer's log files, I made a patch which addressed some problems that appeared to be multi-thread issues. The system kept critical page state in a session object on the server and some user actions would call back to the server for more data or other actions. But the buttons or links that caused these callbacks did not prevent users from double (or triple) clicking and having the same request run on multiple threads. I added some busy indicators and on-click button disabling to prevent multiple concurrent client requests for the same user. Since I couldn’t reproduce the problem in my environment, the hope was that these would also address the data integrity problem, but they did not. 

## A New Year
	
The defect had been in production for at least two months. Returning from the winter holidays it was clear I needed to get more methodical about finding the root cause of this problem. I built another patch with additional logging to try to narrow down where the data loss was happening – on a save? on a page view? internally as data was moving in the application? 
	
On Monday January 9th, we ran a test with the extra logging in place. It showed that the data was reliably moving through the application. The problem was not on saving data; it was on viewing an object. The problem was related to dates, or at least it was most commonly occurred with date values, and when it happened all date fields on the page were lost. 

By this time I was sending daily email updates to the customer.

>
>**Sent: Wednesday, January 11, 7:49 PM**
>
> ...\\
> After saves 3, 8, and 14, the page editor was rendered with no date values – although the values had been saved properly in the database. We learned on Monday that in those situations, if we closed and re-opened the page, the date values would return. \\
> ...

>
>**Sent: Thursday, January 12, 12:17 PM**
> 
><p>This afternoon we ran another test. When the problem occurred, we captured the generated html. Looking at the file, we could see that the generated html (javascript really) included the necessary code to set the proper value. After comparing the text to a properly generated page, the difference is that when we have the problem, we are missing code that brings in certain javascript libraries. For date fields, that code should look like this:</p>
>  {% highlight html %}
<script type="text/javascript">/*<![CDATA[*/app.requires("app_dom", "/solution/scripts/");/*]]>*/</script>
<script type="text/javascript">/*<![CDATA[*/app.requires("app_DatePicker", "/solution/scripts/");/*]]>*/</script>
<div id="container_CASE_X_ACTIVITY_START_DT" class="DatePicker"> ... </div>
{% endhighlight %}
>
>Without these javascript libraries, the value that is written to the page does not get copied to the underlying form field. And if you save, the value is not in the form and so it gets removed. This helps explain why we always lost both dates at the same time. The first date picker thinks it has written the proper script include statements and the second date picker skips that code assuming the first date picker took care of it.
>	
>I am still reading through the code that is responsible for defining these javascript include statements – it is not in solution code per se, it is in one of the java libraries that we use to generate the code for the UI.

## Interlude
	
	// insert long weekend of winter camping

## 	Explaining the Unexplainable

On Tuesday, January 17th, we did another test and confirmed my current hypothesis. The values were being written properly into the HTML. When the problem occurred, the page was missing some of the required Javascript files that needed to be included with the page.
	
I studied the code responsible for generating the include files again. The Java UI component library was complicated. When date picker objects were constructed, they held on to a reference to the HttpRequest and they added themselves into the HttpSession. So the object references looked something like this:

<p><img src="/images/defect/datepicker.references.png" alt="Diagram showing object references" title="Object References" style="max-width:50%;height:auto;"/>.</p>

When each component renders itself onto the http output stream, it first checks with an object in the request to determine if the Javascript include files that the component requires have already been written to the stream or not. If not, it writes the required includes; otherwise it just writes out the code for that specific component.

<p><img src="/images/defect/datepicker.import-tracking.png" alt="Diagram showing import tracking list in the http request" title="Tracking Imports" style="max-width:50%;height:auto;"/>.</p>


The code to render the component seemed straight-forward. (In reality the code was probably in some abstract superclass but the effect is the same.)

{% highlight java %}
public class DateTimePicker {
  ...
  public void render(Writer w) {
	  if ( ! widget.getRequest.scriptsGenerated.includes(jsFilename) ) {
	    this.writeRequiredImportStatement( w, jsFilename )
	    widget.request.scriptsGenerated.add( jsFilename )
	  }
	  w.write ("<div id="%s" class=\"DatePicker\"></div>", id)
  }
}
{% endhighlight %}

The flow looked something like this:
	
... insert picture of sequence diagram ...
	
## toString :-o
	
Since something was causing problems with the import statements, I looked for other callers of the render method and (surprisingly) I found that the standard Java method toString() called render(). toString() is defined on every object in Java and used basically everywhere. 

{% highlight java %}
	public String toString() {
	  StringWriter w = new StringWriter();
	  render( w )
	  return w.toString();
  }
{% endhighlight %}

This meant there were uncountable opportunities for render() to be called. I found that simply putting a breakpoint at the top of the page.render(w) method and inspecting the date picker object would cause the date picker to be rendered on some other stream for the debugger to display but the request would be updated to indicate that the include file had already been generated. :-o While I now knew that any call to toString() could cause the symptoms we were seeing, that didn't explain why we only saw the problem sporadically. We didn't normally connect debuggers to production so any calls to toString() should happen consistently.
	
// To be clear, this was code we were using but not code that my team wrote or owned. In fact, I think it was essentially abandonware by this time. The original authors had gone off to chase some other rainbows so getting a patch would require tracking down someone who had access to the code and understood the library well enough to make a change (not an easy mission at that time).

>**Sent: Tuesday, January 17, 4:43 PM**
	
>...\\
>After learning that the data integrity problem is essentially a problem with the javascript includes, I spent last night and today reviewing the way the UI components generate the html for the required javascript files. From looking at the code I have found ways that the javascript include handler code could fail, but not a way that it could fail only selectively under load. My next step is to try to make another custom patch that would log all calls that manipulate the list of javascript files that have been emitted. I started that update this afternoon but my first build didn’t seem to work, so I am continuing to work on that.\\
>...

## 	Its Always the Last Place You Look 
	
I felt like I was close to finding the problem but could not find a story that explained the symptoms. 
	
There was a very narrow window between when the date picker component was created - really when the list of emitted Javascript files was created - and when the date picker consulted that list to see if the includes had already been written. As far as I knew there was no opportunity for a call to toString() to be made. I had already locked down the system so a user could not have more than one request happening at a time so even though those objects were in the session, other threads should not be accessing the data in the session. The UI components lived in the session while the user was on the page, but they would eventually get cleaned up. Even if old UI components were still in the session, they had already been rendered. What harm could they do now; their request was over. 
	
These appeared to be the facts.
	
- The page components get created, linked to the request, added to the session and then rendered.
- There was a very narrow time window between components being created and subsequently checking the list in the request to determine if the imports need to be written.
- Although the window was pretty narrow, it wasn't *that hard* to cause the problem to occur when there were 100+ user threads all running concurrently
- The UI now prevented the user from initiating multiple concurrent requests that might access their session.
	
It was not possible for a user to get in a situation where the system would invoke toString() on their behalf and corrupt the list of emitted include files. 

As I was writing my daily update email, I can still remember when the sun broke through the clouds. If it was impossible for a user to do it to themselves, then users must somehow do it to each other. "When you have eliminated all which is impossible, then whatever remains, however improbable, must be the truth." Other users wouldn't have direct access to that user's UI components, or to their session. Could a user somehow get hold of the request? Components lived in a session. Sessions lived for a long time. Components held on to their request. What if requests got reused? Lightbulb. If requests are reused, one person's session may hold references to another person's request.
	
It's easy to think of a request as a single use object, but it’s a common optimization for high-priced servers to keep pools of objects to reduce memory usage and garbage collection. When the thought first struck me I had no idea if WebSphere actually reused request objects or not, but I googled around and, sure enough, by default WebSphere reuses requests.
	
What if 2 threads could access the same request at the same time? Then something happening in one user's thread (like calls to toString() that have a side-effect of writing to a request long ago completed) could end up corrupting another user's request. For example:

1. User1 logs in and tries to fill out an incident form. This is Request 1. DatePicker1 and 2 are created and hold onto request 1. DatePicker 1 and 2 are put in User1’s session. The page renders fine.
2. Time passes
3. User 2 logs in and goes to fill in a new case form. This is request 57. Date Picker 113 and 114 are created and hold onto request 57. DatePicker 113 and 114 are put in User2’s session. The page renders fine.
4. Time passes
5. User 1 does something that causes an error email to be issued.
6. User 2 has just saved version 3 of their case form and the system is rendering the updated case page. For this request, Websphere reuses Request 1 for this page.
7. As User 2’s page starts to render, User 1’s error email writes everything in the session to the email. When writing the email, one of those datepickers tells Request 1 that it has successfully written the required date picker Javascript include statement. When User 2’s case page is built, the request indicates that the date picker javascript include has been written already – no need to do that again. User 2’s case page does not have the required Javascript included. The date value is sitting in the generated html but because the Javascript include files are missing, the value does not get copied into the form. If User 2 saves the case, the dates will be removed from the record in the database. Sigh.

Now I had a theory that explained the symptoms. (fist pump emoji)

>**Sent: Thursday, January 19 12:06 AM**
>
>From looking at the code I have found ways that the Javascript include handler code could fail, but not a way that it could fail only selectively under load. 
>
>While I have been typing this email, another possibility occurred to me; one that *is* related to server load. The UI Components like the date picker hold a reference to the request they are created for. They also put themselves in the user’s session, which makes them live longer than the request they were created for. If the request objects are pooled and reused (and in WebSphere at least they are reused) then it would be possible for one of these old date pickers sitting around in a session to cause problems for a new running request, if for example we tried to write the date picker out to a logfile. 
> For tomorrow’s test, I would like us to reconfigure WebSphere on the test environment to not pool request and response objects. 

## Wrap up
	
Sure enough, after changing the WebSphere configuration, we were never able to reproduce the problem again.
	
I don't know what lessons you might take from this story, but the bad decisions I saw in that library stick with me to this day.

1. **toString() had side-effects.**
		Inconceivable 
2. **Objects sat in the session longer than they were needed.**
		This was a consequence of web apps that delegated heavily to the server for behavior, requiring very large session objects to hang onto all that state. They don't build them like they used do - thank god.
3. **Objects kept a reference to their http request**
		Handing the request object to the UI components seemed like a lazy shortcut and an improper distribution of responsibilities. That list of emitted Javascript files should have been in an object representing the page, not directly in the request.
4. **App server error handling explicitly writing out every object in the session.**
		Ironically, the problems would have ceased if we just turned logging off.
		
So there you have it. No fun at the time, but a good story with a happy ending.