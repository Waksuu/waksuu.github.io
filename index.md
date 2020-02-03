# Black box Integration Testing React components connected to Redux

Whats the better way to make your code more maintainable and easier to change in the future than writing tests?
Probably hiring a bunch of interns to test code manually after each commit, but we don't have time and money for that so tests will have to do.

In this post I will focus on the importance of black box testing and how this can be achieved  in React.
I will be using jest, react-testing-liblary and Typescript for my own sanity. I am also gonna assume that you know the basics of these technologies.
In addition I will only focus on Integration Testing (how to test flow of your component without need for backend) because I don't feel that there are good resources for that and Unit Testing is well examined.

#
