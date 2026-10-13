this contains all the kubernetes configs that will be deployed inside the provided infrastructure


The Verdict for your Architecture
Limbo / Proxies (Velocity): Standard Kubernetes Deployment + HPA. They hold no meaningful game state, and players can be instantly rerouted if a pod dies.
-> Proxy probably in the stays just a single pod for the beginning without scaling

Lobbies: Agones Fleet. They hold active players chatting, moving, and interacting. You need Agones to protect them from being killed while players are actively using them.

Minigames / SMP: Agones Fleet + GameServerAllocation. Highly stateful, strict lifecycle management required.