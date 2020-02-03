# Black box Integration Testing React components connected to Redux

Whats the better way to make your code more maintainable and easier to change in the future than writing tests?
Probably hiring a bunch of interns to test code manually after each commit, but we don't have time and money for that so tests will have to do.

In this post I will focus on the importance of black box testing and how this can be achieved  in React.
I will be using jest, react-testing-liblary and Typescript for my own sanity. I am also gonna assume that you know the basics of these technologies.
In addition I will only focus on Integration Testing (how to test flow of your component without need for backend) because I don't feel that there are good resources for that and Unit Testing is well examined.

## Black box testing, why bother?

I am big fan of black box testing, especially when working with front-end. In the world of <i>javascript frameworks chaning so fast that before you download npm dependencies this dependency is already outdated</i> I cannot imagine writing tests that can be easliy broken by the change in implementation that doesn't not have any impact on the behaviour of the application. This kind of enviroment discourages developers from refactoring and writing tests because everything you touch has high change of breaking something (be it tests or the actual code).

We can avoid this problemy by changing the way of thinking about our code and trying to test <i> behaviour </i> and not the internal implementation (that can be easlily changed). Doing so greatly reduces the need of changing tests when we want to change the way that our code works (e.g. refactoring code for optimization reasons).

For example, when testing let's say button counter component, I shouldn't care if the way of storing button counter is implemented within the component as internal state or Redux deals with it, this is an <i>implementation detail</i> that can change, but the  <i> behaviour </i> of the button will stay the same.

You might say, ok this is a very basic exmaple, real world is not that easy and components have some kind of dependencies (e.g component is calling some endpoint to retrieve data), and how do I test it?
