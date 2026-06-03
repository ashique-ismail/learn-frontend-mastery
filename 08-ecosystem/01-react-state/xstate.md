# XState - State Machines and Statecharts

## Overview

XState is a JavaScript/TypeScript library for creating, interpreting, and executing finite state machines and statecharts. It provides a declarative way to model application logic, making complex state management predictable and visualizable.

### Key Features

- **Finite state machines**: Prevent impossible states
- **Statecharts**: Hierarchical and parallel states
- **Actor model**: Spawn and communicate with actors
- **Visualizer**: Visual state machine editor
- **Framework agnostic**: Works with React, Vue, Svelte
- **TypeScript**: Full type safety
- **Testable**: Easy to test state machines

## Installation

```bash
npm install xstate @xstate/react
```

## Basic State Machine

### Simple Toggle Machine

```typescript
import { createMachine, interpret } from 'xstate';
import { useMachine } from '@xstate/react';

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: { TOGGLE: 'active' }
    },
    active: {
      on: { TOGGLE: 'inactive' }
    }
  }
});

function Toggle() {
  const [state, send] = useMachine(toggleMachine);

  return (
    <button onClick={() => send('TOGGLE')}>
      {state.matches('active') ? 'ON' : 'OFF'}
    </button>
  );
}
```

### Traffic Light Machine

```typescript
import { createMachine } from 'xstate';
import { useMachine } from '@xstate/react';

const trafficLightMachine = createMachine({
  id: 'trafficLight',
  initial: 'red',
  states: {
    red: {
      after: {
        3000: 'green'
      }
    },
    green: {
      after: {
        2000: 'yellow'
      }
    },
    yellow: {
      after: {
        1000: 'red'
      }
    }
  }
});

function TrafficLight() {
  const [state] = useMachine(trafficLightMachine);

  return (
    <div>
      <div style={{ 
        width: 100, 
        height: 100, 
        borderRadius: '50%',
        backgroundColor: state.value as string
      }} />
      <p>Current: {state.value}</p>
    </div>
  );
}
```

## Context and Actions

### Counter with Context

```typescript
import { createMachine, assign } from 'xstate';
import { useMachine } from '@xstate/react';

interface CounterContext {
  count: number;
}

type CounterEvent =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET'; value: number };

const counterMachine = createMachine<CounterContext, CounterEvent>({
  id: 'counter',
  initial: 'active',
  context: {
    count: 0
  },
  states: {
    active: {
      on: {
        INCREMENT: {
          actions: assign({
            count: (context) => context.count + 1
          })
        },
        DECREMENT: {
          actions: assign({
            count: (context) => context.count - 1
          })
        },
        RESET: {
          actions: assign({
            count: 0
          })
        },
        SET: {
          actions: assign({
            count: (_, event) => event.value
          })
        }
      }
    }
  }
});

function Counter() {
  const [state, send] = useMachine(counterMachine);

  return (
    <div>
      <p>Count: {state.context.count}</p>
      <button onClick={() => send('INCREMENT')}>+</button>
      <button onClick={() => send('DECREMENT')}>-</button>
      <button onClick={() => send('RESET')}>Reset</button>
      <button onClick={() => send({ type: 'SET', value: 10 })}>Set to 10</button>
    </div>
  );
}
```

## Guards (Conditional Transitions)

```typescript
import { createMachine, assign } from 'xstate';

const gatedCounterMachine = createMachine({
  id: 'gatedCounter',
  context: { count: 0 },
  initial: 'counting',
  states: {
    counting: {
      on: {
        INCREMENT: {
          actions: assign({
            count: (ctx) => ctx.count + 1
          }),
          cond: (ctx) => ctx.count < 10
        },
        DECREMENT: {
          actions: assign({
            count: (ctx) => ctx.count - 1
          }),
          cond: (ctx) => ctx.count > 0
        }
      }
    }
  }
});
```

## Async Services

### Fetching Data

```typescript
import { createMachine, assign } from 'xstate';
import { useMachine } from '@xstate/react';

interface User {
  id: string;
  name: string;
  email: string;
}

interface DataContext {
  user: User | null;
  error: string | null;
}

type DataEvent =
  | { type: 'FETCH' }
  | { type: 'RETRY' };

const fetchMachine = createMachine<DataContext, DataEvent>({
  id: 'fetch',
  initial: 'idle',
  context: {
    user: null,
    error: null
  },
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      invoke: {
        src: async () => {
          const response = await fetch('/api/user');
          if (!response.ok) throw new Error('Failed to fetch');
          return response.json();
        },
        onDone: {
          target: 'success',
          actions: assign({
            user: (_, event) => event.data,
            error: null
          })
        },
        onError: {
          target: 'failure',
          actions: assign({
            error: (_, event) => (event.data as Error).message,
            user: null
          })
        }
      }
    },
    success: {
      on: { FETCH: 'loading' }
    },
    failure: {
      on: { 
        RETRY: 'loading',
        FETCH: 'loading'
      }
    }
  }
});

function DataFetcher() {
  const [state, send] = useMachine(fetchMachine);

  return (
    <div>
      {state.matches('idle') && (
        <button onClick={() => send('FETCH')}>Fetch User</button>
      )}
      {state.matches('loading') && <p>Loading...</p>}
      {state.matches('success') && state.context.user && (
        <div>
          <h3>{state.context.user.name}</h3>
          <p>{state.context.user.email}</p>
          <button onClick={() => send('FETCH')}>Refresh</button>
        </div>
      )}
      {state.matches('failure') && (
        <div>
          <p>Error: {state.context.error}</p>
          <button onClick={() => send('RETRY')}>Retry</button>
        </div>
      )}
    </div>
  );
}
```

## Hierarchical States

```typescript
import { createMachine } from 'xstate';

const authMachine = createMachine({
  id: 'auth',
  initial: 'unauthenticated',
  states: {
    unauthenticated: {
      initial: 'idle',
      states: {
        idle: {
          on: { LOGIN: 'loggingIn' }
        },
        loggingIn: {
          invoke: {
            src: 'login',
            onDone: '#auth.authenticated',
            onError: 'error'
          }
        },
        error: {
          on: { LOGIN: 'loggingIn' }
        }
      }
    },
    authenticated: {
      initial: 'idle',
      states: {
        idle: {},
        refreshing: {
          invoke: {
            src: 'refreshToken',
            onDone: 'idle',
            onError: '#auth.unauthenticated.error'
          }
        }
      },
      on: {
        LOGOUT: 'unauthenticated'
      }
    }
  }
});
```

## Parallel States

```typescript
import { createMachine } from 'xstate';

const mediaMachine = createMachine({
  id: 'media',
  type: 'parallel',
  states: {
    audio: {
      initial: 'muted',
      states: {
        muted: {
          on: { UNMUTE: 'unmuted' }
        },
        unmuted: {
          on: { MUTE: 'muted' }
        }
      }
    },
    playback: {
      initial: 'paused',
      states: {
        paused: {
          on: { PLAY: 'playing' }
        },
        playing: {
          on: { PAUSE: 'paused' }
        }
      }
    }
  }
});

function MediaPlayer() {
  const [state, send] = useMachine(mediaMachine);

  return (
    <div>
      <button onClick={() => send('PLAY')}>
        {state.matches({ playback: 'playing' }) ? 'Pause' : 'Play'}
      </button>
      <button onClick={() => send('MUTE')}>
        {state.matches({ audio: 'muted' }) ? 'Unmute' : 'Mute'}
      </button>
      <p>Audio: {state.matches({ audio: 'muted' }) ? 'Muted' : 'Unmuted'}</p>
      <p>Playback: {state.matches({ playback: 'playing' }) ? 'Playing' : 'Paused'}</p>
    </div>
  );
}
```

## History States

```typescript
const workflowMachine = createMachine({
  id: 'workflow',
  initial: 'edit',
  states: {
    edit: {
      initial: 'text',
      states: {
        text: {
          on: { TO_IMAGE: 'image' }
        },
        image: {
          on: { TO_TEXT: 'text' }
        },
        hist: {
          type: 'history'
        }
      },
      on: {
        PREVIEW: 'preview'
      }
    },
    preview: {
      on: {
        BACK: 'edit.hist' // Returns to last edit state
      }
    }
  }
});
```

## Actor Model

```typescript
import { createMachine, spawn, assign } from 'xstate';
import { useMachine } from '@xstate/react';

const todoMachine = createMachine({
  id: 'todo',
  initial: 'active',
  context: {
    text: '',
    completed: false
  },
  states: {
    active: {
      on: {
        TOGGLE: {
          target: 'completed',
          actions: assign({ completed: true })
        }
      }
    },
    completed: {
      on: {
        TOGGLE: {
          target: 'active',
          actions: assign({ completed: false })
        }
      }
    }
  }
});

const todosMachine = createMachine({
  id: 'todos',
  context: {
    todos: []
  },
  on: {
    'ADD_TODO': {
      actions: assign({
        todos: (context, event) => [
          ...context.todos,
          spawn(todoMachine.withContext({ 
            text: event.text,
            completed: false 
          }))
        ]
      })
    }
  }
});
```

## Common Patterns

### Form Machine

```typescript
const formMachine = createMachine({
  id: 'form',
  initial: 'editing',
  context: {
    formData: {},
    errors: {}
  },
  states: {
    editing: {
      on: {
        SUBMIT: {
          target: 'validating',
          actions: assign({
            formData: (_, event) => event.data
          })
        }
      }
    },
    validating: {
      invoke: {
        src: async (context) => {
          // Validation logic
          const errors = {};
          if (!context.formData.email) {
            errors.email = 'Required';
          }
          if (Object.keys(errors).length > 0) {
            throw errors;
          }
          return context.formData;
        },
        onDone: 'submitting',
        onError: {
          target: 'editing',
          actions: assign({
            errors: (_, event) => event.data
          })
        }
      }
    },
    submitting: {
      invoke: {
        src: async (context) => {
          const response = await fetch('/api/submit', {
            method: 'POST',
            body: JSON.stringify(context.formData)
          });
          return response.json();
        },
        onDone: 'success',
        onError: {
          target: 'editing',
          actions: assign({
            errors: { submit: 'Failed to submit' }
          })
        }
      }
    },
    success: {
      type: 'final'
    }
  }
});
```

## Testing

```typescript
import { createMachine } from 'xstate';
import { interpret } from 'xstate';

describe('toggle machine', () => {
  it('should toggle states', (done) => {
    const toggleMachine = createMachine({
      initial: 'inactive',
      states: {
        inactive: { on: { TOGGLE: 'active' } },
        active: { on: { TOGGLE: 'inactive' } }
      }
    });

    const service = interpret(toggleMachine)
      .onTransition((state) => {
        if (state.matches('active')) {
          done();
        }
      })
      .start();

    service.send('TOGGLE');
  });
});
```

## Common Mistakes

1. **Not using guards**: Missing conditional logic
2. **Overcomplicating machines**: Keep simple when possible
3. **Not handling all events**: Missing event handlers
4. **Ignoring context**: Not using context for state data
5. **Not visualizing**: Missing the visualizer benefits

## Best Practices

1. **Use TypeScript**: Full type safety for events and context
2. **Visualize machines**: Use XState visualizer
3. **Keep machines focused**: Single responsibility
4. **Use hierarchical states**: For complex logic
5. **Test state machines**: Easy to test deterministically
6. **Document with statecharts**: Self-documenting code
7. **Use actors**: For spawning independent machines

## When to Use XState

### Good Use Cases
- Complex workflows and wizards
- Authentication flows
- Form validation
- Multi-step processes
- Game logic
- Protocol implementations

### Not Ideal For
- Simple toggle states
- Basic data fetching
- When team is unfamiliar with state machines

## Interview Questions

1. **What is a finite state machine?**
   - A model with finite states and transitions between them

2. **What are statecharts?**
   - Extended state machines with hierarchical and parallel states

3. **What is context in XState?**
   - Extended state data associated with the machine

4. **What are guards?**
   - Conditions that determine if a transition should occur

5. **What is the actor model?**
   - Pattern for spawning and managing independent state machines

## Key Takeaways

- XState provides declarative state management
- Prevents impossible states
- Visualizable and testable
- Hierarchical and parallel states
- Actor model for complex systems
- Framework agnostic
- Excellent TypeScript support
- Self-documenting with statecharts

## Resources

- [Official Documentation](https://xstate.js.org/)
- [Visualizer](https://stately.ai/viz)
- [Examples](https://xstate.js.org/docs/examples/)
- [GitHub](https://github.com/statelyai/xstate)
