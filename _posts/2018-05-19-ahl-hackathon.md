---
layout: post
title: AHL Python Hackathon April 2018
---

Three weeks ago MAN AHL organised an [opensource
hackathon](https://www.ahl.com/hackathon) at their London office. As part of the
Hackathon people should contribute to one of the PyData artifacts they regularly
use. To support them in making their first contribution, AHL also coordinated
that several core committers of opensource projects were present at the event. I
joined in as the representative of the [Apache Arrow](https://arrow.apache.org/)
project.

While it was the first of a kind hackathon for me, it was also the first event
for the Apache Arrow project where we had to do preparations for a set of new
contributors joining at once. This meant that we had to do some homework in the
project. Before the hackathon, the [JIRA
issues](https://issues.apache.org/jira/projects/ARROW) were mostly a mixture of
larger topics we planned to work on or small todos that surfaced during a pull
request review. For a successful hackathon we needed to define more tasks that
were in the 1-2h or the 10h range. This should then allow people to work on at
least a small and a medium task at the hackathon.

The event itself was located at AHL's London office at Riverbank House. We had a
nice view over the Thames during the hackathon. The first day started with
introductory talks that detailed how the event is structured and why a company
like AHL is hosting it. Their CTO explained that while the hackathon will
definitely be a boost for their brand in the space of searching for new
developers it is also a great opportunity to give something back to the Python
ecosystem that they rely upon. (One must also note at this point that they are
also the hosts for the PyData London meetup group.) Afterwards each of the core
committers presented their open source project and detailed some of the
contribution possibilities for the weekend. After a splendid lunch, people
distributed to the available projects and then started hacking on their
contributions.

Hacking continued for 24 hours with superb meals in between (they even had
organized some late night Sushi). People happily made their first small open
source contributions and then continued to tackle slightly larger issues. At the
end of the hackathon we had final presentations by each of the participating
projects about what they have achieved. The most amazing of all was awarded with
a small price (some retro consoles): [One team worked in a group of four with
nearly no sleep to implement Stable distribution in
scipy.](http://shape-of-code.coding-guidelines.com/2018/04/23/a-python-data-science-hackathon/)

During the hackathon, the team Apache Arrow solved various small and medium
issues. It consisted of [Kee Chong Tan](https://github.com/keechongtan), [Donal
Simmie](https://github.com/dsimmie), [Gatis Seja](https://github.com/Gatisseja),
[Samuel Sinayoko](https://github.com/samuelsinayoko), and [Florian
Rathgeber](https://github.com/kynan). To get started, we got [improved table
column access](https://github.com/apache/arrow/pull/1923), [column selection in
`from_pandas`](https://github.com/apache/arrow/pull/1924) and [a better
documentation on
`pyarrow.parquet.write_dataset`](https://github.com/apache/arrow/pull/1925) for
Arrow. People tripped over some small issues in Apache Arrow while they were
trying to implement their features. I used this to directly
[fix](https://github.com/apache/arrow/pull/1927)
[them](https://github.com/apache/arrow/pull/1926).

After the small ones passed the code review and CI builds, they were
successfully merged to master. Then people progressed to [construct Arrow (and
Pandas) schemas from DataFrames without converting it fully to an Arrow table
first](https://github.com/apache/arrow/pull/1929), [expose the C++ Builder
classes in Python](https://github.com/apache/arrow/pull/1930), [add zero copy
returns for plain NumPy array without Pandas
specifics](https://github.com/apache/arrow/pull/1931), and improve the usability
and documentation of the combination of Arrow and Numba. 

Looking back at the hackathon, it was a great event. I have met new contributors
and existing users that have different use cases for Apache Arrow than what we
use a it for. The hackathon also showed clearly what you need to get new
contributors on board: a nice elevator pitch why they should use your project, a
decent set of documentation and also a nicely prepared list of possible tasks.
Personally, I have also learnt on how to mentor people to do their first
contributions and what are the main hurdles to get there. The hackathon was
really a win-win situation for the project and the people that came for making
their first contributions. They got support and in-person feedback while
developing new features whereas the project itself gained a lot by also getting
in-person feedback on how hard it to join the group of committers. In the end
this resulted for me in a long list of homework on how to improve the experience
for people that are interested in making their first contribution to Apache
Arrow.

While we should continously update and spread our elevator pitch, we also need
to take care to write tutorials for people that are new to the project. For
example we have a clear specification of the memory layout but lack a
comprehensive guide that does not go into alll detail but outlines the essential
features of this memory layout. Furthermore Arrow is a really complicated
project. For the Python implementation, you often need to touch Python, Cython
and C++ code. We need a small introduction on how and when these three languages
are used and where to look at for a certain functionality.

The final thing to make the project more successful for new contributors: list a
lot of small tasks somewhere that are easy to implement. While you will build up
good relationships with people as long as you have a welcoming and suppotive
community, many people will only browse your issues to find a task they could
work on. You won't have the chance to be talk to them and help them make a
contribution unless they can find a task they could talk to you about.
