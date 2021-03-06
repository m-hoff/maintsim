# Simantha

Simantha uses discrete event simulation to model the behavior of discrete manufacturing systems, in particular the production and maintenance functions mass production systems. 

## Using this package

### Requirements

Simantha requires Python &ge; 3.6 and [SciPy](https://www.scipy.org/) &ge; 1.5.2 for running tests.

### Installation

Install from PyPi using

```
pip install simantha
```

### Setting up a manufacturing system

A system consists of various assets that are linked together by a specified routing. The available assets are described below.

All asset classes take an optional `name` parameter which is useful for identifying unique objects in the simulation trace. 

#### Source

`simantha.Source`

A source introduces parts to the system. Each source should be directly upstream of a machine. 

Parameters
- `interarrival_time` - time between arrivals generated by the source. If an arrival occurs but there is no available space for the part, the new arrival will be seized as soon as space becomes available. By default, the interarrival time is zero, meaning machines downstream of the source will never be starved. 

#### Machine

`simantha.Machine`

While a machine operates it will continuously take parts from an upstream container (if the container is not empty), process the part, and place the part in a downstream container (if the container is not full). Machines can be subject to periodic degradation and failure, at which point they will require maintenance action before they can be restored.

Currently, machines may only follow a Markovian degradation process.<sup>[1](#chan)</sup> Under this degradation mode, a degradation transition matrix is specified as a parameter of a machine. State transitions of this Markov process represent changes in the degradation level of the machine. In general, it is assumed that there is one absorbing state that represents machine failure. 

Parameters
- `cycle_time` - the duration of time a machine must process a part before it can be placed at the next station.
- `degradation_matrix` - the degradation transition matrix for Markovian degradation. For an `n`x`n` transition matrix, degradation levels are integer-labeled from `0`, indicating perfect health, to `n-1`, indicating the machine is failed. 
- `cbm_threshold` - the condition-based maintenance threshold for creating a request for a maintenance resource. For an `n`x`n` degradation transition matrix, this threshold should be in the interval `[0, n-1]`.
- `pm_distribution` - random distribution of the duration of any preventive maintenance job performed on a machine. Any maintenenance that is performed on a machine that is not in the failed state is considered preventive. 
- `cm_distribution` - random distribution of the duration of any corrective maintenance job performed on a machine. A machine in a failed state can only be serviced by corrective maintenance. 

Currently, only a uniform random distrubtion is supported and should take the form `{'uniform': [a, b]}` when passed as an argument to a `Machine`. 

#### Buffer

`simantha.Buffer`

Buffers serve as passive containers for parts that are not being processed by a machine. Generally, each machine should retrieve parts from one buffer (or a source) and place parts in one buffer (or a sink). If machines are assigned more than one immediate upstream or downstream buffer unexpected transfer behavior can occur. 

Parameters
- `capacity` - the maximum number of parts that may occupy a buffer at one time.

#### Sink

`simantha.Sink`

A sink collects finished parts that exit the system. Each part that is placed in a sink is counted as a part that has been successfully produced by the system. The total level of all sinks is considered to be the overall system production. 

Parameters
- `initial_level` - the number of finished parts in a sink when the system is initialized. Sinks are intialized as empty by default. 

#### Maintainer

`simantha.Maintainer`

A maintainer is responsible for repairing machines that have either failed or requested maintenance. If the number of machines requesting maintenance exceeds the maintainer's available capacity, the default behavior is to maintain the machine that requested maintenance the earilest (a first-come, first-serve policy). This behvaior can be modified by overriding the `choose_maintenance_action` method of the `simantha.Maintainer` class. This method takes the current `queue` in the form of a list of `simantha.Machine` instances requesting maintenance as an argument and should return the instance that is assigned maintenance. 

Parameters
- `capacity` - the maximum number of machines that can be maintained simultaneously.

#### Defining object routing

Once the objects are created the routing(s) through the system need to be defined using each asset's `define_routing` method. This method takes two arguments, `upstream` and `downstream`, as lists of other objects in the system directly adjacent to the object calling the method.


#### System

`simantha.System`

All of the objects are passed to a `System` object that can then be simulated.

Parameters
- `objects` - a list of objects including sources, machines, buffers, and sinks that make up the system.
- `maintainer` - an instance of `Maintainer` if one has been created. If one has not been created and machines are subject to degradation, a maintainer with the default first-in, first-out policy will be used. 

### Simulating a system

A `simantha.System` object has two methods for simulating its behavior:
- `simulate` - simulates system behavior for the specified duration of time. Arguments of this method include
  - `warm_up_time` - the duration of time the system operation is simulated before performance statistics are gathered. Since objects in the system are empty by default, specifying a warm up period will allow the system to reach steady state performance which is often of interest when conducting any analysis.
  - `simulation_time` - the duration of time to simulate system behavior. Performance statistics will be gathered during this period. In total, the duration of simulated time is `warm_up_time` + `simulation_time`.
  - `verbose` - `True` or `False`, indicating whether a summary of the simulation run should be displayed. `True` by default.
  - `collect_data` - `True` or `False`, indicating whether or not data is collected for indiviudial objects in the system. If many simulation runs are conducted, setting this to `False` may improve performance. 
- `iterate_simulation` - conduct multiple simulation runs of a system. Useful for estimating the average performance of a particular system whose behavior is random. This method uses Python's [multiprocessing](https://docs.python.org/3.8/library/multiprocessing.html) to call the `simulate` method in parallel. Arguments to this method are
  - `replications` - the number of simulation runs to conduct. 
  - `warm_up_time` - used the same as in the `simulate` method and applied to each replication.
  - `simulation_time` - used the same as in the `simulate` method.
  - `verbose` - `True` or `False`, indicating whether or not a summary of all replications is displayed at completion. `True` by default.
  - `jobs` - the number of worker processes that will be created by the multiprocessing module. By default, one worker is used which is the equivalent of running all replications in series. 

### Example usage

Below is a simple example demonstrating the production of a single machine.

```python
>>> from simantha import Source, Machine, Sink, System
>>> 
>>> source = Source()
>>> M1 = Machine('M1', cycle_time=1)
>>> sink = Sink()
>>>
>>> source.define_routing(downstream=[M1])
>>> M1.define_routing(upstream=[source], downstream=[sink])
>>> sink.define_routing(upstream=[M1])
>>> 
>>> system = System(objects=[source, M1, sink])
>>> system.simulate(simulation_time=100)
Simulation finished in 0.00s
Parts produced: 100
```

As a slightly more complex example, the code below constructs a two-machine one-buffer line where each machine is subject to degradation. The degradation transition matrix will represent Bernoulli reliability with p = 0.01.<sup>[2](#li)</sup> Additionally, the maintainer capacity is set to 1, indicating that only one machine may be repaired at a time.

```python
>>> from simantha import Source, Machine, Buffer, Sink, System, Maintainer
>>> 
>>> degradation_matrix = [[0.99, 0.01], [0., 1.]]
>>> cm_distribution = {'uniform': [10, 20]}
>>> 
>>> source = Source()
>>> M1 = Machine(
...     'M1', 
...     cycle_time=1, 
...     degradation_matrix=degradation_matrix,
...     cm_distribution=cm_distribution,
... )
>>> B1 = Buffer(capacity=5)
>>> M2 = Machine(
...     'M2',
...     cycle_time=1,
...     degradation_matrix=degradation_matrix,
...     cm_distribution=cm_distribution
... )
>>> sink = Sink()
>>> 
>>> source.define_routing(downstream=[M1])
>>> M1.define_routing(upstream=[source], downstream=[B1])
>>> B1.define_routing(upstream=[M1], downstream=[M2])
>>> M2.define_routing(upstream=[M2], downstream=[sink])
>>> sink.define_routing(upstream=[M2])
>>> 
>>> maintainer = Maintaner(capacity=1)
>>> 
>>> system = System(objects=[source, M1, B1, M2, sink], maintainer=maintainer)
>>> system.simulate(simulation_time=1000)
Simulation finished in 0.05s
Parts produced: 748
```

Lastly, the following example will show a system consisting of two machines in parallel. Both `M1` and `M2` retrieve parts from the same source and place finished parts in the same sink.

```python
>>> from simantha import Source, Machine, Sink, System
>>> 
>>> source = Source()
>>> M1 = Machine('M1', cycle_time=1)
>>> M2 = Machine('M2', cycle_time=1)
>>> sink = Sink()
>>> 
>>> source.define_routing(downstream=[M1, M2])
>>> for machine in [M1, M2]:
...     machine.define_routing(upstream=[source], downstream=[sink])
... 
>>> sink.define_routing(upstream=[M1, M2])
>>> 
>>> system = System(objects=[source, M1, M2, sink])
>>> system.simulate(simulation_time=100)
Simulation finished in 0.01s
Parts produced: 200
```

The elements of these examples can be used to more complex configurations of arbitrary structure. Additionally, the `Machine` class may represent other operations in a manufacturing system, such as an inspection station or a material handling process. 

Additional examples are located in `simantha/examples/`.

## References

<a name="chan">1.</a> Chan, G. K., & Asgarpoor, S. (2006). Optimum maintenance policy with Markov processes. Electric power systems research, 76(6-7), 452-456. https://doi.org/10.1016/j.epsr.2005.09.010

<a name="li">2.</a> Li, J., & Meerkov, S. M. (2008). Production systems engineering. Springer Science & Business Media. https://doi.org/10.1007/978-0-387-75579-3