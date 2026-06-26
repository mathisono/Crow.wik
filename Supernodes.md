# Supernodes

Crow can run on an AREDN supernode to relay Crow/MeshIP traffic between participating AREDN meshes.

This page is carried forward from the original Raven supernode guidance, with Crow-specific implementation notes added from the current code.

## What problem supernodes solve

A normal Crow node shares traffic inside its local AREDN mesh. It does not automatically make every other AREDN mesh part of the same Crow message space.

To share Crow traffic with other AREDN meshes, install Crow on the local AREDN supernode path. Crow instances on participating supernodes advertise themselves through AREDN service discovery and relay Crow/MeshIP traffic between meshes that also have Crow on their supernode side.

## Installing on a supernode

Install Crow on the supernode the same way as a normal AREDN node.

No separate Crow-only supernode switch is needed for basic operation. Crow reads the AREDN supernode setting from the node configuration and automatically changes behavior when it detects that it is running on a supernode.

## What Crow changes automatically

When Crow starts on AREDN, the platform code checks the AREDN supernode flag. If the node is a supernode, Crow treats it as a relay-only instance.

On a supernode, Crow automatically ignores or removes these configuration areas at runtime:

- `meshtastic`
- `meshcore`
- `textstore`
- `messages`

That means a Crow supernode does **not** act as a Meshtastic bridge, MeshCore bridge, node message store, or RAM-message client.

## Restrictions

When Crow is installed on a supernode:

1. Meshtastic configuration is ignored.
2. MeshCore configuration is ignored.
3. Text store / node message store configuration is ignored.
4. Message RAM configuration is ignored.
5. The supernode should not be used as a persistent message archive.
6. You may technically send messages from the supernode's Crow UI, but operationally it is better to treat the supernode as infrastructure, not as a user station.

## In use

A Crow supernode relays Crow/MeshIP traffic from the local AREDN mesh toward other AREDN meshes. This only works end-to-end where the other mesh also has a Crow supernode instance or a Crow/MeshIP forwarder path.

The supernode advertises MeshIP bridge capability so normal Crow nodes can discover a path beyond the local mesh. Normal nodes then see the supernode path as a relay, while message storage and user-facing operations stay on local nodes.

## Recommended topology

Use this pattern:

```text
Local AREDN mesh
├── Normal Crow node with USB storage
│   ├── local channels
│   ├── node message store / textstore
│   ├── images and shared files
│   └── Winlink forms
└── Crow on AREDN supernode
    └── Crow/MeshIP relay to other participating meshes
```

This keeps the supernode focused on routing/relay work and keeps persistent history on a normal node that has known storage, reliable power, and local administrative control.

## Do not put the message store on the supernode

The original Raven guidance explicitly avoided text stores on supernodes, because a supernode could otherwise end up trying to store traffic for many remote networks. Crow keeps the same rule in code: when running on a supernode, `textstore` config is ignored.

For message history, use a normal node with USB storage enabled. For supernode relay, use the supernode.

## Checklist

After installation:

1. Confirm the AREDN node is actually configured as a supernode.
2. Install Crow normally.
3. Restart Crow.
4. Do **not** add Meshtastic, MeshCore, or textstore configuration to the supernode.
5. Put message-store duties on a normal Crow node.
6. Test relay by using Crow nodes on both sides of the participating supernode path.
