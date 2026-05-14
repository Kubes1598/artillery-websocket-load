# artillery-websocket-load

An Artillery scenario that opens and sustains 5,000 concurrent Socket.IO connections. Each client sends heartbeats, joins rooms, and broadcasts messages — the way a real chat or notifications app behaves.

Request/response load tools (JMeter, k6) only exercise the WebSocket handshake. Real-time apps fail differently. Connection limits at the OS layer. Broadcast amplification that scales worse than linearly. Pub/sub backlog when fanout outpaces consumption. None of those show up in a handshake test.

## What it does

- Opens 5,000 long-lived Socket.IO connections, ramping over 5 minutes then sustaining for another 5.
- Each client sends a heartbeat every 10 seconds.
- 20% of clients are "chatty" — they emit broadcast messages on a random interval.
- Periodic room join/leave to model real-world churn.
- Server-side metrics scraped by Prometheus in parallel so latency on the client side correlates with CPU and event-loop lag on the server.

## A real run

When I pointed this at a Socket.IO + Redis app, three things showed up that no other load test would have caught:

- Around 2,100 connected clients, Redis pub/sub backlog started growing past 50ms.
- Around 3,400, the broadcast fanout cost went O(n²). A single message cost 11.5MB of allocations.
- At 5,000, event-loop lag exceeded 200ms and new connections began failing.

The fixes ended up being a per-room fan-out worker and a connection shard by tenant. Both were verified by re-running this exact scenario until the numbers settled.

## Run it

```bash
npm install
npx artillery run ws-load.yml
```

For a real test you'll want to point it at staging, not localhost. The `--target` flag overrides the URL in the YAML.

## Companion

- Server-side observability that makes this useful: [prometheus-grafana-stack](https://github.com/Kubes1598/prometheus-grafana-stack).
- Portfolio case study: [ajimati-portfolio](https://github.com/Kubes1598/ajimati-portfolio).

## License

MIT. See [LICENSE](./LICENSE).
