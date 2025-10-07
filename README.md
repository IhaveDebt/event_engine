/**
 * Event Sourcing Engine (event_engine.ts)
 *
 * Lightweight event store with emit, subscribe and replay.
 *
 * Run:
 *   ts-node src/event_engine.ts
 */

import crypto from 'crypto';

type Event = { id: string; type: string; data: any; timestamp: number };
type EventHandler = (event: Event) => void;

class EventStore {
  private events: Event[] = [];
  private handlers: Record<string, EventHandler[]> = {};

  emit(type: string, data: any) {
    const event: Event = { id: crypto.randomUUID(), type, data, timestamp: Date.now() };
    this.events.push(event);
    (this.handlers[type] || []).forEach(h => {
      try {
        h(event);
      } catch (e) {
        console.error('Handler error', e);
      }
    });
  }

  on(type: string, handler: EventHandler) {
    if (!this.handlers[type]) this.handlers[type] = [];
    this.handlers[type].push(handler);
  }

  replay(filter?: (e: Event) => boolean) {
    console.log("Replaying events...");
    for (const e of this.events) {
      if (!filter || filter(e)) {
        (this.handlers[e.type] || []).forEach(h => {
          try { h(e); } catch (err) { console.error('Replay handler error', err); }
        });
      }
    }
  }

  list() {
    return this.events.slice();
  }
}

// Demo usage when run directly
if (require.main === module) {
  const store = new EventStore();

  store.on('deposit', e => console.log(`[handler] Deposited $${e.data.amount} (evt ${e.id})`));
  store.on('withdraw', e => console.log(`[handler] Withdrew $${e.data.amount} (evt ${e.id})`));

  store.emit('deposit', { amount: 100 });
  store.emit('withdraw', { amount: 40 });
  store.emit('deposit', { amount: 25 });

  console.log('Events stored:', store.list().length);
  store.replay();
}
