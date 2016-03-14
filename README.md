# musings-on-distributed-ci-cd
musings on distributed CI/CD


#### Are CI/CD pipeline dependencies recursive?

*Pseudo BNF*
```
manifest             ::= repository hash
manifests            ::= manifest | manifest, manifests
tool-manifest        ::= repository hash
tool-manifests       ::= tool-manifest | tool-manifest, tool-manifests
build-manifest       ::= manifest, tool-manifest
artifact             ::= object

build                ::= manifests tool-manifests, artifact, artifact-hash
build-artifact       ::= build configuration build-manifest
build-artifacts      ::= build configuration build-manifest | build-artifact, build-artifacts
build-artifact-hash  ::= hash(build-artifacts), configuration

deploy-spec          ::= replicationcontroller, service, load-balancer, secrets
deploy-target        ::= zone, cluster, namespace, build-artifacts, deploy-spec
deploy-chat-opts     ::= /deploy deploy-target deploy-artifact-hash, configuration
deploy               ::= build-artifact, deploy-target

promote-target       ::= zone, cluster, namespace, build-artifact
promotion-chat-ops   ::= /promote promote-target build-artifact-hash, configuration
promote              ::= promotion-action, deploy-spec, deploy-target
```


#### Build system design

Build systems are essentially built around solving their graphing
problems.

Ignoring the bootstrapping problem of initializing and OS or first
order build tools, let's look at a simple view of the description of
CI/CD as a graphing problem.

```
item <- depends on artifact a
item <- a
item -> build results in artifact a
item -> a

d' depends on d
c depends on d' and b'
c depends on d',d,b',b,a',a

d' <- d

c <- b' 
c <- d'

a -> a'
b -> b'

c <- a'
c <- b'

c <- b
c <- a

      + d'<-d
     /
c <-+
     \
      + b'<-b <- a' <- a

```

There's some utility in this form of expression.

0. It describes inherent ordering and access
  - The build dependencies, sequences and available parallelism
  - The deploy dependencies, sequences and available parallelism

For example:
  - the build of d' can run in parallel or follow the execution of
    either of
    - a -> a'
    - b -> b'
  - the build of c requires both b' and d'
  - the build of c can't begin until the completion of b' and d'

Traditional tooling like make, cmake, ant, maven, distmake, jenkins,
travis, gitlab have varying levels of tooling around this area.

e.g. make's makefile syntax describes dependencies and resolves the
DAG with cycles identified and removed or errors.

#### Build Deploy and Beyond
Notation:

```
a depends on b
a <- b
B(x) build component x
D(x) deploy component x
C(x) deploy config generation x
```


```

B(a) <- a,B(b)

D(a) <- D(c),B(a),D(d)

D(c) <- C(d)

D(a) <- D(c) <- B(c)
     <- D(d)

D(d) <- B(d) <- d

Build Graph

       + a
      /
B(a) <-+
      \
       + B(b) <- b


Deploy Dependency Graph 

            
                     + a        
                    /           
          +   B(a) <-+           
         /          \           
        /            + B(b) <- b
       /
      / 
D(a) <- 
      \
       \
        + D(c) <- B(c) <- c
        |     \
        |      \
        |       \             
        + D(d) <-+ C(d) <- B(d) <- d
                             
```

Parallelism available in build/deploy graph

```

chat-ops
              parallel                     parallel             
           fan out   fan in          fan out        fan in      
             + B(b)    +                +              +        
            /           \              /                \       
/build   ->+   B(c)      +-> B(a) -> -+   D(c)         -+-> D(a)
            \           /  \      /    \                /       
             + B(d)    +    +C(d)+      + D(d)         +        
                             
```
