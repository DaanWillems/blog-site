---
title: "Rigid body dynamics 2D"
date: "2024-02-01"
summary: "Detecting collisions between rigid bodies"
toc: false
readTime: false
autonumber: true
math: true
katex: true
tags: ["physics2d", "rust"]
showTags: false
---

Recently I got really into the idea of building a 2D physics engine myself. The idea isn't very practical as physics engine are complex, and I am certainly not going to build one that's very good. However I thought it might be insightful to try anyway. 

When talking about a physics engine. I am talking a about a computer program that tries to simulate the interaction between physics objects. A common application for physics engines are video games. They make sure that the player character does not fall through the ground, or can walk through walls. Fancy video games may also model behaviours that items move when pushed. 

The nice thing about building a physics engine for a game is that it does not have to be perfectly realistic. Somewhat believable will do just fine. This makes the task ahead of us much easier, but still quite hard. 

One of the first issues I ran into is that while there are many resources for learning about physics engines, they all focus on slightly different parts of the implemention. Some are more concerned with collision detection, while others with resolving collisions. There were actually very little sources that gave a full overview of all components. 

Then there were also many different interpretations of what a physics engine actually is, ranging from very simplified implementations to wildly complex mathematics. By scrouncing together many different blog posts, comments, youtube video's and scientific papers I managed to create an implementation I am happy with. In the following posts I'll try and walk you through building one like I did, so that the next person trying to learn will have one more additional post to read. 

I split my implementations roughtly into three parts. Collision detection, collision resolution and stabilisation. In this post I will go over some of the basics of simulating bodies. In part 2 I will discuss a method of implementing collision resolution by impulses. Finally part 3 discusses various stabilisation techniques to improve the quality of the simulation

# Some basic physics
A physics engine used in 2D video games should model the behaviour of two dimensional objects and how they interact with each other. To create an acceptable and simple simulation, we only need to use so called rigid bodies. Rigid bodies are solid bodies that cannot deform. For most games rigid bodies are sufficient to create a satisfactory simulation.

We'll go over how to simulate the movement of bodies through space on a computer.

## The three laws of motion
Rigid bodies move through space, affected by external forces such as gravity, collisions and other constraints we might invent. Their motion can be modeled using Newtonian mechanics, which are founded on Isaac Newton's Three Laws of Motion:

**1. Inertia**
A body remains at rest, or in motion at a constant speed in a straight line, except insofar as it is acted upon by a force.

**2. F = ma**
At any instant of time, the net force on a body is equal to the body's acceleration multiplied by its mass or, equivalently, the rate at which the body's momentum is changing with time.

**3. Action and reaction**
If two bodies exert forces on each other, these forces have the same magnitude but opposite directions. For every reaction there is an equal and opposite reaction.

With these laws we can simulate realistic movement. Let's apply these rules on one of the most simple objects we can think of: particles.

## Particles
Before simulating an actual rigid body, we can start with particles. Using particles greatly simplifies the simulation as they are 'shapeless'. They can be understood as a point which has position in space, a velocity and mass.

### Simulating motion
The first thing to understand is how we can simulate simple movement through space on a computer. This is already fundementally different than in the real world, as objects simulated by a computer can only move in discrete steps. 

To start out simple we will simulate a shapeless and massless particle. This way we don't have to account for gravity. 

TODO: Insert picture of particle


### Position, velocity and acceleration

In order to simulate the particle moving through space, we need to define some of the properties it has. To know where it is currently, our particle should have a position (pos) property which we can represent with a two dimensional vector.

$$ pos = \begin{bmatrix} x \\\\ y \end{bmatrix} $$

The particle also has a velocity (v), which represents the direction the particle is travelling towards, and the speed it has. In other words, the velocity describes how the position changes over time. Again this can be defined with a two dimensional vector.

$$ v = \begin{bmatrix} x \\\\ y \end{bmatrix} $$

We can use these properties to determine the position of a particle at a specific point in time, given that we have the original position and the direction and magnitude of the velocity. 

Imagine the particle's starting position is at the origin and is now moving towards the right (1, 0) of the screen with a speed of 5. We know that that at: t = 0 the position of the particle is (0, 0) and the velocity is (5, 0). The acceleration is (0, 0) because the speed is constant, therefore we can ignore it. If we wanted to find where the particle is at t = 5 we could define a mathematical function that describes the position over time, and solve it for t = 5

$$ pos(t) = \begin{bmatrix} 0 \\\\ 0 \end{bmatrix} + \begin{bmatrix} 5 \\\\ 0 \end{bmatrix} \cdot t $$

Plugging in t = 5 gives 

$$ pos(t) = \begin{bmatrix} 5 \\\\ 0 \end{bmatrix} \cdot 5  =  \begin{bmatrix} 25 \\\\ 0 \end{bmatrix}$$

We can see that if we assume the velocity is static, it's easy to find the exact position of the particle at any given point in time.

As mentioned, the velocity is the rate of change of position of an object. By defining a function that gives the position over time, we have actually calculated the integral of the velocity vector over time:

$$ pos(t) = \int{} v \,dt = v \cdot t + C $$

(Where C represents the starting position. In this case (0, 0))

In the example, the velocity was constant. However a simulation without acceleration is not too interesting. Let's see how the acceleration fits together with the formula's above.

We determined that we can find the function describing the position of a body by taking the integral of the velocity vector over time. Much like velocity describes the rate of change of the position. The acceleration describes the rate of change of the velocity over time. Therefore taking the integral of the acceleration function will yield the velocity function.

This displays the fundamental relationship between the acceleration, velocity and position. 

$$ a = \dot{v} = \ddot{pos} $$

In other words: The velocity is the derivative of the position, and the acceleration is the second derivative of the position. 

Given the above, you would maybe think that simulating the particle is easy! Simply solve the equations for a given acceleration value, and find the position of the particle at any point in time. Indeed, taking the acceleration function and finding the position through analytical integration is one possible approach of simulating the particle. However, this approach quickly falls apart when you desire a more complex simulation, where the acceleration is also unknown, and must be inferred from multiple forces.

It is possible to sacrifice some of the accuracy analytical integration provides, and in turn reduce the complexity of the calculations. The following section describes how to achieve this with numerical integration.

# Numerical integration
Solving integrals that are very hard is a recurring problem in computer science. When an integral becomes too complex to solve analytically (like in the chapter above). It can make sense to instead try and step through the function by picking numbers for the variables, and increasing them a little with each step. By summing up the resulting values from the functions, we can find the total area under the function. In the case of the velocity function, this are corresponds to the distance traveled, therefore to the position.

By stepping through the function we are essentially approximating the distance travelled, instead of getting an exact answer by solving the integral analytically. This mean's we have traded away some accuracy, but have obtained a much simpler method of finding the position. This solution still provides an answer that is accuracy enough to base our simulation on.

We'll look at explicit euler integration, as it is the most simple form of numerical integration. With euler integration we can find the numerical value by 'stepping through' the function in small timesteps. This is much simpler to implement than solving the integral. Let's consider our velocity function.

$$ v = \begin{bmatrix} x \\\\ y \end{bmatrix} $$

If we have position of a particle $ p_0 $ at $ t = 0 $, and we wanted to know the position $p_1$ of the particle at $t = 1$. We can do:

$$ p_1 = p_0 + v $$

```
acc = (0, 9.81)
vel = vel + acc
pos = pos + vel
```

In the above example we set the acceleration to the force of gravity, each timestep we add the acceleration to the velocity, and the resulting velocity to the position. As time between the timesteps decreases, the computation becomes more accurate. Of course we cannot make the timestep infinitely small, we have to make some compromises. 

![Integration table](/images/integration-table.png)

TODO: Compare numerical integration accuracy vs analytical for a falling particle with a nice figure

## Particle simulation algorithm
Using the methods described above it's possible to implement a simple simulation with particles. The implementation below is written in Rust using the Bevy game engine to render shapes and manage state. 

First define the relevant properties for particles


```rust
#[derive(Component)]
pub struct Position(pub Vec2);

#[derive(Component)]
pub struct Velocity(pub Vec2);
```

If a player presses 'space' a new particle is spawned with these two properties, and also a mesh particle which is necessary to render the shape.

```rust
if keys.just_pressed(KeyCode::Space) {
    if let Some(world_position) = window
        .cursor_position()
        .and_then(|cursor| camera.viewport_to_world_2d(camera_transform, cursor))
    {
        let shape = Mesh::from(Circle::new(20.));
        let color = ColorMaterial::from(Color::rgb(1., 0.4, 0.));

        let mesh_handle = meshes.add(shape);
        let material_handle = materials.add(color);
        commands.spawn((
            Position(world_position),
            Velocity(Vec2::new(0., 0.)),
            MaterialMesh2dBundle {
                mesh: mesh_handle.into(),
                material: material_handle,
                ..default()
            },
        ));
    }
}
```

In order to simulate movement an integration step is added. All entities with a velocity and position component are queried and updated. The integration is done with explicit euler integration as described above.

```rust
pub fn integrate(
    mut commands: Commands,
    time: Res<Time>,
    mut query: Query<(
        Entity,
        &mut Velocity,
        &mut Position,
    )>,
) {
    for (entity, mut velocity, mut position) in
        query.iter_mut()
    {
        //Despawn the entity if it falls out of the screen
        if position.0.y < -1000. {
            commands.entity(entity).despawn();
        }

        //If the velocity becomes sufficiently small, just set it to 0
        if velocity.0.length() < 0.1 {
            velocity.0.x = 0.;
            velocity.0.y = 0.;
        }

        //Apply gravity
        velocity.0.y -= 9.81;

        position.0 += velocity.0 * time.delta_seconds();
    }
}
```

In order to get the particle to render, some additional methods are necessary. The project_positions system takes the data from position component and mutates the mesh accordingly. The system spawn_camera makes sure a window/viewport appears.

```rust
pub fn spawn_camera(mut commands: Commands, mut windows: Query<&mut Window>) {
    commands.spawn((Camera2dBundle::default(), MainCamera));
    let mut window = windows.single_mut();
    window.resolution.set(2000., 1000.);
}
```

These systems are tied together in a Bevy app together with some boilerplate code to get a window.

```rust
fn main() {
App::new()
    .add_plugins(DefaultPlugins)
    .add_systems(Startup, (render::spawn_camera))
    .add_systems(
        Update,
        (
            input_handler.before(integrate),
            integrate.before(render::project_positions),
            render::project_positions.after(integrate),
        ),
    )
    .run();
}
```

To view the finished program, go to: ..

If we compile the program and run it we should get an empty screen. Pressing space spawns a particle which quickly falls outside of the screen. All appears to be working!

# Rigid bodies
Having considered a simple particle simulation, we can attempt to try to simulate rigid bodies. Rigid bodies represent shapes that cannot be deformed. This makes them much more simple to simulate, as we do not have to account for the deformations when they interact. 

As opposed to particles, rigid bodies do actually have a shape. In this article we will only consider mathematically simple shapes, such as rectangles and circles. 

In order to make a simulation that tries to represent reality, we should consider the additional properties a rigid body has. Applying a force to the object affects it differently depending on where the force is applied. To determine the desired response to applying force to a body, we have to consider the center of mass and the moment of inertia. 

## Center of mass
From Newton's second law of motion follows the equation:

$$ F = ma $$

Which means: The force, _F_ equals the mass times the acceleration. If we multiply the mass by the velocity, we get the 'linear momentum' often denoted by _p_. The force can also be seen as the derivative of the linear momentum.

$$ F = \dot{p} = m\dot{v} = ma $$

The particle just simulated had mass, but because it had no shape the mass was all 'contained' within in an infinitely small point. This allowed us to find the acceleration by dividing the force by the mass. 

Because rigid bodies do have a shape, the mass is distributed around the object. We can think of the body a a bunch of points, where each point has a little mass. All the points combined form the total mass of the object. Therefore the total linear momentum of the body is the sum of the linear momentum of the all the mass points summed. 

$$ p^n = \sum_n{m^i \cdot v^i} $$

<!-- If we use the above equations to find the acceleration of an object after applying a force to it, we would end up with:

$$ a = \frac{F}{m} =  $$ -->

This makes reasoning about the mass much more complex, however because rigid bodies cannot deform. We can simplify their mass by calculating a center of mass. 

We can sum all the vectors to each little point of mass in the object multiplied by its position, and divide the resulting vector by the total mass of the object. This gives us a vector pointing to the center of mass of a body. 

$$ CoM = \frac{\sum_n{m^i \cdot v^i}}{M} $$

The object naturally rotates around it's center of mass. In our simulation, we will consider the the position of the body to be equivelant to the position of the center of mass. In this article I'll only consider bodies that have perfectly uniform mass. 

If we want to reason about a how a force affects the body, we can now assume that any force acting upon it is directly and only acting upon the center of mass, and the entire body moves together with the center of mass.

## Angular



