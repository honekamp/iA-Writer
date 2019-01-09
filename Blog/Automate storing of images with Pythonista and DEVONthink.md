One of my habits is to take a screenshot of my iPhone and iPad home screen on the first day of each month and store the screenshots away for further reference.

Until very recently, I used to store the screenshots somewhere in the file system.

I used to have a [*Workflow*][wf] to store the screenshots into their respective directories. But at some point, the workflow broke on the basis of no longer being able to access an arbitrary directory in response to stricter sandboxing rules. 

As a remedy to this issue, I considered adding the screenshots to one of my [DEVONthink (To Go)][dt] databases. In this case, I would no longer have a problem with sandboxing and can still add the screenshots to separate folders **inside** of the *DEVONthink* database.

However, to make this possible I need to automate the process of adding of an image to *DEVONthink*. And this can - to the best of my knowledge - only be done by using the URL scheme for *DEVONthink*.

To celebrate the [news][forum] that [*Pythonista*][omz-py] is back in active development, I spontaneously decided to try to implement the storing of screenshots to *DEVONthink* using *Pythonista*[^5].

I learned a lot during this quest and found surprisingly little documentation about *DEVONthink*‘s URL scheme. Therefore, I figured it might be worth sharing my experience for the potential benefit of fellow *DEVONthink* users with a similar goal.

I started with one of the example scripts that implements access to the screenshots in my *Photos* library and modified it accordingly.

```python
album = photos.get_screenshots_album()
screenshots = album.assets
if not screenshots:
    print('You have no screenshots in your library.')
else:
    # Access latest screenshot
    newest = screenshots[-1]
```

I learned that *Photos* provides a „virtual album“ of all screenshots in your library and that this album is accessible (you guessed it) by calling the method `photos.get_screenshots_album()`. 

This call returns an `AssetCollection`, of which the last element (an `Asset`) is taken into further consideration in the form of the variable `newest`.

After - by way of accessing the element with the index -1 - achieving access to the latest screenshot, meta-data (specifically the creation date) are utilized to form the title of the screenshot in the *DEVONthink* database.

In order to not break the pattern of naming the existing screenshots, the title takes the form of a filename using a {yyyy}_{MM}.{imagetype} pattern[^2], even if it is stored in a *DEVONthink* database.

The year can be accessed by calling the `creation_date.year` property on  `newest`.

```python
    year = newest.creation_date.year
    month = '{0:0>2}'.format(newest.creation_date.month)
    title='{0}_{1}.png'.format(year,month)
```

Setting the month was a bit of a challenge because the pattern asks for a two-digit month and the property `creation_date.month` returns just a single digit for all single-digit months.

Of course, this issue could be solved "quick and dirty". But I wanted to find a solution by using `str.format()` with a fitting format string[^4]. The result is shown in the code snippet above. I'm not sure whether this is the most elegant solution, but at least it works.

As mentioned before, I did not find any official documentation of the URL Scheme for *DEVONthink*, especially for the creation of images. After a round of searching, I found some hints in an article Federico Viticci at [Macstories][ms]. That got me started. 

Also, Gabe Weatherhead over at [Macdrifter][md] has documented some of his automation (mostly for adding markdown text) of *DEVONthink*.

The usage of any URL Scheme to pass an image to the database requires the serialization[^3] of the image in a URL-friendly form:

```python
    img = newest.get_image_data(original=False)
    b64encoded=base64.b64encode(img.getvalue())
    urlEncodedPic = urllib.parse.quote_plus(b64encoded)
```

In addition, it is also necessary to decide about the target folder to store the image. As mentioned before, screenshots for iPhone are stored in a different folder than screenshots for iPad.

In other words, it will be necessary to programatically find out whether the script runs on iPad or iPhone and select the target folder accordingly. To solve this issue, I posted a [question to the *Pythonista* User Forum][omz-forum] and quickly got a response: use `platform.machine()`.

I've had an eye on this method before, but unfortunately was distracted by the [documentation][pf] that mentioned 'i386' as a possible return value and that (at least in my personal understanding) suggests a hint about the architecture rather than the concrete hardware.

A deeper look at the chapter in the documentation might have revealed that there is also `platform.architecture()` and therefore `platform.machine()` very likely has a different purpose than to return machine's architecture. Anyway ...

```python
    device = platform.machine()
    if (device.startswith('iPad')):
      destination = '58B06234-37B8-485F-AB7C-DC67CACFB9FB'
    elif(device.startswith('iPhone')):
      destination = '71E35B25-466E-4E2A-AE2D-C65C4BAFB254'
```

This finally gets the last ingredient for the creation of the URL and its subsequent execution.

```python
    url = 'x-devonthink://createimage?source={0}&title={1}&destination={2}'.format(urlEncodedPic,title,destination)
    webbrowser.open(url)
``` 

The argument named `source` passes the serialized image. The call to `webbrowser.open()` finally launches *DEVONthink* and adds the screenshot to the respective folder.

I have to say that this really works quite well and I'm very pleased with the result. Of course, the amount of time spent working on the implementation of the discussed script would probably be good for manually storing the screenshots worth of several years. 

But that's not the point. I learned a lot in the process and I may now be in a better position to solve further automation tasks with less effort. For example, variations of the script to handle photos or text rather than screenshots could easily be created.

[forum]: https://forum.omz-software.com/topic/5295/new-beta-for-pythonista-3-3
[wf]: http://workflow.is/
[sc]: https://scriptable.app
[dt]: https://www.devontechnologies.com/products/devonthink/overview.html
[omz-py]: http://omz-software.com/pythonista/
[md]: https://macdrifter.com
[ms]: https://www.macstories.net/ios/ipad-diaries-devonthinks-new-advanced-automation/
[omz-forum]: https://forum.omz-software.com/topic/5329/programatically-retrieve-device-on-which-the-script-is-running
[pf]: https://docs.python.org/3/library/platform.html?highlight=platform#module-platform

[^2]: Where {yyyy} represents the four-digit year and {MM} represents the two digit month, with January being represented by '01'. The {imagetype} boils down to 'png' or 'jpeg'.

[^3]: This means that the image is transformed into a sequence of textual characters form which the receiving end can reconstruct the original image.

[^4]: I'm a big fan of string formatting.

[^5]: A possible alternative would have been to do the implementation in Javascript using [*Scriptable*][sc] as the development platform. However, I know almost nothing about Javascript. My Python is better, but still not on expert level.