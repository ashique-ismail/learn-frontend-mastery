# Headless UI

## The Idea

**In plain English:** Headless UI is a library of ready-made interactive building blocks (like dropdowns, modals, and toggles) that handle all the behind-the-scenes logic and keyboard controls for you, but come with zero built-in visual styling — you decide exactly how they look.

**Real-world analogy:** Think of buying a brand-new car chassis from a manufacturer. The chassis comes with a working engine, steering, brakes, and safety systems already built in — but no paint, no seats, no body panels. You bolt on whatever custom body and interior you want.

- The chassis with engine and brakes = Headless UI components (behavior, keyboard navigation, accessibility)
- The unpainted, unstyled shell = no default visual styles applied
- Your custom paint, seats, and panels = the Tailwind CSS classes you add yourself

---

## Introduction

Headless UI is a completely unstyled, fully accessible UI component library, designed to integrate beautifully with Tailwind CSS. Created by the Tailwind Labs team, it provides React and Vue components that handle all the complex behavior, keyboard interactions, focus management, and accessibility while leaving styling entirely up to you.

## Installation and Setup

### Installation

```bash
npm install @headlessui/react

# or with yarn
yarn add @headlessui/react

# or with pnpm
pnpm add @headlessui/react
```

### With Tailwind CSS

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```js
// tailwind.config.js
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## Core Components

### Menu (Dropdown)

```tsx
import { Menu, Transition } from '@headlessui/react';
import { Fragment } from 'react';
import { ChevronDownIcon } from '@heroicons/react/20/solid';

function MyDropdown() {
  return (
    <Menu as="div" className="relative inline-block text-left">
      <div>
        <Menu.Button className="inline-flex w-full justify-center gap-x-1.5 rounded-md bg-white px-3 py-2 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50">
          Options
          <ChevronDownIcon className="-mr-1 h-5 w-5 text-gray-400" aria-hidden="true" />
        </Menu.Button>
      </div>

      <Transition
        as={Fragment}
        enter="transition ease-out duration-100"
        enterFrom="transform opacity-0 scale-95"
        enterTo="transform opacity-100 scale-100"
        leave="transition ease-in duration-75"
        leaveFrom="transform opacity-100 scale-100"
        leaveTo="transform opacity-0 scale-95"
      >
        <Menu.Items className="absolute right-0 z-10 mt-2 w-56 origin-top-right rounded-md bg-white shadow-lg ring-1 ring-black ring-opacity-5 focus:outline-none">
          <div className="py-1">
            <Menu.Item>
              {({ active }) => (
                <a
                  href="#"
                  className={`${
                    active ? 'bg-gray-100 text-gray-900' : 'text-gray-700'
                  } block px-4 py-2 text-sm`}
                >
                  Account settings
                </a>
              )}
            </Menu.Item>
            <Menu.Item>
              {({ active }) => (
                <a
                  href="#"
                  className={`${
                    active ? 'bg-gray-100 text-gray-900' : 'text-gray-700'
                  } block px-4 py-2 text-sm`}
                >
                  Support
                </a>
              )}
            </Menu.Item>
            <Menu.Item>
              {({ active }) => (
                <a
                  href="#"
                  className={`${
                    active ? 'bg-gray-100 text-gray-900' : 'text-gray-700'
                  } block px-4 py-2 text-sm`}
                >
                  License
                </a>
              )}
            </Menu.Item>
            <form method="POST" action="#">
              <Menu.Item>
                {({ active }) => (
                  <button
                    type="submit"
                    className={`${
                      active ? 'bg-gray-100 text-gray-900' : 'text-gray-700'
                    } block w-full px-4 py-2 text-left text-sm`}
                  >
                    Sign out
                  </button>
                )}
              </Menu.Item>
            </form>
          </div>
        </Menu.Items>
      </Transition>
    </Menu>
  );
}

export default MyDropdown;
```

### Dialog (Modal)

```tsx
import { Dialog, Transition } from '@headlessui/react';
import { Fragment, useState } from 'react';
import { XMarkIcon } from '@heroicons/react/24/outline';

function MyModal() {
  const [isOpen, setIsOpen] = useState(false);

  function closeModal() {
    setIsOpen(false);
  }

  function openModal() {
    setIsOpen(true);
  }

  return (
    <>
      <button
        type="button"
        onClick={openModal}
        className="rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2"
      >
        Open dialog
      </button>

      <Transition appear show={isOpen} as={Fragment}>
        <Dialog as="div" className="relative z-10" onClose={closeModal}>
          <Transition.Child
            as={Fragment}
            enter="ease-out duration-300"
            enterFrom="opacity-0"
            enterTo="opacity-100"
            leave="ease-in duration-200"
            leaveFrom="opacity-100"
            leaveTo="opacity-0"
          >
            <div className="fixed inset-0 bg-black bg-opacity-25" />
          </Transition.Child>

          <div className="fixed inset-0 overflow-y-auto">
            <div className="flex min-h-full items-center justify-center p-4 text-center">
              <Transition.Child
                as={Fragment}
                enter="ease-out duration-300"
                enterFrom="opacity-0 scale-95"
                enterTo="opacity-100 scale-100"
                leave="ease-in duration-200"
                leaveFrom="opacity-100 scale-100"
                leaveTo="opacity-0 scale-95"
              >
                <Dialog.Panel className="w-full max-w-md transform overflow-hidden rounded-2xl bg-white p-6 text-left align-middle shadow-xl transition-all">
                  <Dialog.Title
                    as="h3"
                    className="text-lg font-medium leading-6 text-gray-900"
                  >
                    Payment successful
                  </Dialog.Title>
                  <button
                    type="button"
                    className="absolute right-4 top-4 text-gray-400 hover:text-gray-500"
                    onClick={closeModal}
                  >
                    <XMarkIcon className="h-6 w-6" aria-hidden="true" />
                  </button>
                  <div className="mt-2">
                    <p className="text-sm text-gray-500">
                      Your payment has been successfully submitted. We've sent
                      you an email with all of the details of your order.
                    </p>
                  </div>

                  <div className="mt-4">
                    <button
                      type="button"
                      className="inline-flex justify-center rounded-md border border-transparent bg-blue-100 px-4 py-2 text-sm font-medium text-blue-900 hover:bg-blue-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 focus-visible:ring-offset-2"
                      onClick={closeModal}
                    >
                      Got it, thanks!
                    </button>
                  </div>
                </Dialog.Panel>
              </Transition.Child>
            </div>
          </div>
        </Dialog>
      </Transition>
    </>
  );
}

export default MyModal;
```

### Listbox (Select)

```tsx
import { Listbox, Transition } from '@headlessui/react';
import { Fragment, useState } from 'react';
import { CheckIcon, ChevronUpDownIcon } from '@heroicons/react/20/solid';

const people = [
  { id: 1, name: 'Wade Cooper' },
  { id: 2, name: 'Arlene Mccoy' },
  { id: 3, name: 'Devon Webb' },
  { id: 4, name: 'Tom Cook' },
  { id: 5, name: 'Tanya Fox' },
  { id: 6, name: 'Hellen Schmidt' },
];

function MyListbox() {
  const [selected, setSelected] = useState(people[0]);

  return (
    <div className="w-72">
      <Listbox value={selected} onChange={setSelected}>
        <div className="relative mt-1">
          <Listbox.Button className="relative w-full cursor-default rounded-lg bg-white py-2 pl-3 pr-10 text-left shadow-md focus:outline-none focus-visible:border-indigo-500 focus-visible:ring-2 focus-visible:ring-white focus-visible:ring-opacity-75 focus-visible:ring-offset-2 focus-visible:ring-offset-orange-300 sm:text-sm">
            <span className="block truncate">{selected.name}</span>
            <span className="pointer-events-none absolute inset-y-0 right-0 flex items-center pr-2">
              <ChevronUpDownIcon
                className="h-5 w-5 text-gray-400"
                aria-hidden="true"
              />
            </span>
          </Listbox.Button>
          <Transition
            as={Fragment}
            leave="transition ease-in duration-100"
            leaveFrom="opacity-100"
            leaveTo="opacity-0"
          >
            <Listbox.Options className="absolute mt-1 max-h-60 w-full overflow-auto rounded-md bg-white py-1 text-base shadow-lg ring-1 ring-black ring-opacity-5 focus:outline-none sm:text-sm">
              {people.map((person) => (
                <Listbox.Option
                  key={person.id}
                  className={({ active }) =>
                    `relative cursor-default select-none py-2 pl-10 pr-4 ${
                      active ? 'bg-amber-100 text-amber-900' : 'text-gray-900'
                    }`
                  }
                  value={person}
                >
                  {({ selected }) => (
                    <>
                      <span
                        className={`block truncate ${
                          selected ? 'font-medium' : 'font-normal'
                        }`}
                      >
                        {person.name}
                      </span>
                      {selected ? (
                        <span className="absolute inset-y-0 left-0 flex items-center pl-3 text-amber-600">
                          <CheckIcon className="h-5 w-5" aria-hidden="true" />
                        </span>
                      ) : null}
                    </>
                  )}
                </Listbox.Option>
              ))}
            </Listbox.Options>
          </Transition>
        </div>
      </Listbox>
    </div>
  );
}

export default MyListbox;
```

### Combobox (Autocomplete)

```tsx
import { Combobox, Transition } from '@headlessui/react';
import { Fragment, useState } from 'react';
import { CheckIcon, ChevronUpDownIcon } from '@heroicons/react/20/solid';

const people = [
  { id: 1, name: 'Wade Cooper' },
  { id: 2, name: 'Arlene Mccoy' },
  { id: 3, name: 'Devon Webb' },
  { id: 4, name: 'Tom Cook' },
  { id: 5, name: 'Tanya Fox' },
];

function MyCombobox() {
  const [selected, setSelected] = useState(people[0]);
  const [query, setQuery] = useState('');

  const filteredPeople =
    query === ''
      ? people
      : people.filter((person) =>
          person.name
            .toLowerCase()
            .replace(/\s+/g, '')
            .includes(query.toLowerCase().replace(/\s+/g, ''))
        );

  return (
    <div className="w-72">
      <Combobox value={selected} onChange={setSelected}>
        <div className="relative mt-1">
          <div className="relative w-full cursor-default overflow-hidden rounded-lg bg-white text-left shadow-md focus:outline-none focus-visible:ring-2 focus-visible:ring-white focus-visible:ring-opacity-75 focus-visible:ring-offset-2 focus-visible:ring-offset-teal-300 sm:text-sm">
            <Combobox.Input
              className="w-full border-none py-2 pl-3 pr-10 text-sm leading-5 text-gray-900 focus:ring-0"
              displayValue={(person: typeof people[0]) => person.name}
              onChange={(event) => setQuery(event.target.value)}
            />
            <Combobox.Button className="absolute inset-y-0 right-0 flex items-center pr-2">
              <ChevronUpDownIcon
                className="h-5 w-5 text-gray-400"
                aria-hidden="true"
              />
            </Combobox.Button>
          </div>
          <Transition
            as={Fragment}
            leave="transition ease-in duration-100"
            leaveFrom="opacity-100"
            leaveTo="opacity-0"
            afterLeave={() => setQuery('')}
          >
            <Combobox.Options className="absolute mt-1 max-h-60 w-full overflow-auto rounded-md bg-white py-1 text-base shadow-lg ring-1 ring-black ring-opacity-5 focus:outline-none sm:text-sm">
              {filteredPeople.length === 0 && query !== '' ? (
                <div className="relative cursor-default select-none py-2 px-4 text-gray-700">
                  Nothing found.
                </div>
              ) : (
                filteredPeople.map((person) => (
                  <Combobox.Option
                    key={person.id}
                    className={({ active }) =>
                      `relative cursor-default select-none py-2 pl-10 pr-4 ${
                        active ? 'bg-teal-600 text-white' : 'text-gray-900'
                      }`
                    }
                    value={person}
                  >
                    {({ selected, active }) => (
                      <>
                        <span
                          className={`block truncate ${
                            selected ? 'font-medium' : 'font-normal'
                          }`}
                        >
                          {person.name}
                        </span>
                        {selected ? (
                          <span
                            className={`absolute inset-y-0 left-0 flex items-center pl-3 ${
                              active ? 'text-white' : 'text-teal-600'
                            }`}
                          >
                            <CheckIcon className="h-5 w-5" aria-hidden="true" />
                          </span>
                        ) : null}
                      </>
                    )}
                  </Combobox.Option>
                ))
              )}
            </Combobox.Options>
          </Transition>
        </div>
      </Combobox>
    </div>
  );
}

export default MyCombobox;
```

### Switch (Toggle)

```tsx
import { Switch } from '@headlessui/react';
import { useState } from 'react';

function MyToggle() {
  const [enabled, setEnabled] = useState(false);

  return (
    <Switch
      checked={enabled}
      onChange={setEnabled}
      className={`${enabled ? 'bg-blue-600' : 'bg-gray-200'}
        relative inline-flex h-6 w-11 shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none focus-visible:ring-2  focus-visible:ring-white focus-visible:ring-opacity-75`}
    >
      <span className="sr-only">Use setting</span>
      <span
        aria-hidden="true"
        className={`${enabled ? 'translate-x-5' : 'translate-x-0'}
          pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow-lg ring-0 transition duration-200 ease-in-out`}
      />
    </Switch>
  );
}

export default MyToggle;
```

### Disclosure (Accordion)

```tsx
import { Disclosure } from '@headlessui/react';
import { ChevronUpIcon } from '@heroicons/react/20/solid';

function MyDisclosure() {
  return (
    <div className="w-full px-4 pt-16">
      <div className="mx-auto w-full max-w-md rounded-2xl bg-white p-2">
        <Disclosure>
          {({ open }) => (
            <>
              <Disclosure.Button className="flex w-full justify-between rounded-lg bg-purple-100 px-4 py-2 text-left text-sm font-medium text-purple-900 hover:bg-purple-200 focus:outline-none focus-visible:ring focus-visible:ring-purple-500 focus-visible:ring-opacity-75">
                <span>What is your refund policy?</span>
                <ChevronUpIcon
                  className={`${
                    open ? 'rotate-180 transform' : ''
                  } h-5 w-5 text-purple-500`}
                />
              </Disclosure.Button>
              <Disclosure.Panel className="px-4 pt-4 pb-2 text-sm text-gray-500">
                If you're unhappy with your purchase for any reason, email us
                within 90 days and we'll refund you in full, no questions asked.
              </Disclosure.Panel>
            </>
          )}
        </Disclosure>
        <Disclosure as="div" className="mt-2">
          {({ open }) => (
            <>
              <Disclosure.Button className="flex w-full justify-between rounded-lg bg-purple-100 px-4 py-2 text-left text-sm font-medium text-purple-900 hover:bg-purple-200 focus:outline-none focus-visible:ring focus-visible:ring-purple-500 focus-visible:ring-opacity-75">
                <span>Do you offer technical support?</span>
                <ChevronUpIcon
                  className={`${
                    open ? 'rotate-180 transform' : ''
                  } h-5 w-5 text-purple-500`}
                />
              </Disclosure.Button>
              <Disclosure.Panel className="px-4 pt-4 pb-2 text-sm text-gray-500">
                No.
              </Disclosure.Panel>
            </>
          )}
        </Disclosure>
      </div>
    </div>
  );
}

export default MyDisclosure;
```

### Tab Group

```tsx
import { Tab } from '@headlessui/react';

function classNames(...classes: string[]) {
  return classes.filter(Boolean).join(' ');
}

function MyTabs() {
  const categories = {
    Recent: [
      { id: 1, title: 'Does drinking coffee make you smarter?', date: '5h ago' },
      { id: 2, title: "So you've bought coffee... now what?", date: '2h ago' },
    ],
    Popular: [
      { id: 1, title: 'Is tech making coffee better or worse?', date: 'Jan 7' },
      { id: 2, title: 'The most innovative things happening in coffee', date: 'Mar 19' },
    ],
    Trending: [
      { id: 1, title: 'Ask Me Anything: 10 answers to your questions about coffee', date: '2d ago' },
      { id: 2, title: "The worst advice we've ever heard about coffee", date: '4d ago' },
    ],
  };

  return (
    <div className="w-full max-w-md px-2 py-16 sm:px-0">
      <Tab.Group>
        <Tab.List className="flex space-x-1 rounded-xl bg-blue-900/20 p-1">
          {Object.keys(categories).map((category) => (
            <Tab
              key={category}
              className={({ selected }) =>
                classNames(
                  'w-full rounded-lg py-2.5 text-sm font-medium leading-5',
                  'ring-white ring-opacity-60 ring-offset-2 ring-offset-blue-400 focus:outline-none focus:ring-2',
                  selected
                    ? 'bg-white text-blue-700 shadow'
                    : 'text-blue-100 hover:bg-white/[0.12] hover:text-white'
                )
              }
            >
              {category}
            </Tab>
          ))}
        </Tab.List>
        <Tab.Panels className="mt-2">
          {Object.values(categories).map((posts, idx) => (
            <Tab.Panel
              key={idx}
              className={classNames(
                'rounded-xl bg-white p-3',
                'ring-white ring-opacity-60 ring-offset-2 ring-offset-blue-400 focus:outline-none focus:ring-2'
              )}
            >
              <ul>
                {posts.map((post) => (
                  <li
                    key={post.id}
                    className="relative rounded-md p-3 hover:bg-gray-100"
                  >
                    <h3 className="text-sm font-medium leading-5">
                      {post.title}
                    </h3>
                    <ul className="mt-1 flex space-x-1 text-xs font-normal leading-4 text-gray-500">
                      <li>{post.date}</li>
                    </ul>
                  </li>
                ))}
              </ul>
            </Tab.Panel>
          ))}
        </Tab.Panels>
      </Tab.Group>
    </div>
  );
}

export default MyTabs;
```

### Popover

```tsx
import { Popover, Transition } from '@headlessui/react';
import { Fragment } from 'react';
import { ChevronDownIcon } from '@heroicons/react/20/solid';

function MyPopover() {
  return (
    <div className="w-full max-w-sm px-4">
      <Popover className="relative">
        {({ open }) => (
          <>
            <Popover.Button
              className={`${open ? 'text-white' : 'text-white/90'}
                group inline-flex items-center rounded-md bg-orange-700 px-3 py-2 text-base font-medium hover:text-white focus:outline-none focus-visible:ring-2 focus-visible:ring-white focus-visible:ring-opacity-75`}
            >
              <span>Solutions</span>
              <ChevronDownIcon
                className={`${open ? 'text-orange-300' : 'text-orange-300/70'}
                  ml-2 h-5 w-5 transition duration-150 ease-in-out group-hover:text-orange-300/80`}
                aria-hidden="true"
              />
            </Popover.Button>
            <Transition
              as={Fragment}
              enter="transition ease-out duration-200"
              enterFrom="opacity-0 translate-y-1"
              enterTo="opacity-100 translate-y-0"
              leave="transition ease-in duration-150"
              leaveFrom="opacity-100 translate-y-0"
              leaveTo="opacity-0 translate-y-1"
            >
              <Popover.Panel className="absolute left-1/2 z-10 mt-3 w-screen max-w-sm -translate-x-1/2 transform px-4 sm:px-0 lg:max-w-3xl">
                <div className="overflow-hidden rounded-lg shadow-lg ring-1 ring-black ring-opacity-5">
                  <div className="relative grid gap-8 bg-white p-7 lg:grid-cols-2">
                    <a
                      href="#"
                      className="-m-3 flex items-center rounded-lg p-2 transition duration-150 ease-in-out hover:bg-gray-50 focus:outline-none focus-visible:ring focus-visible:ring-orange-500 focus-visible:ring-opacity-50"
                    >
                      <div className="ml-4">
                        <p className="text-sm font-medium text-gray-900">
                          Insights
                        </p>
                        <p className="text-sm text-gray-500">
                          Measure actions your users take
                        </p>
                      </div>
                    </a>
                    <a
                      href="#"
                      className="-m-3 flex items-center rounded-lg p-2 transition duration-150 ease-in-out hover:bg-gray-50 focus:outline-none focus-visible:ring focus-visible:ring-orange-500 focus-visible:ring-opacity-50"
                    >
                      <div className="ml-4">
                        <p className="text-sm font-medium text-gray-900">
                          Automations
                        </p>
                        <p className="text-sm text-gray-500">
                          Create your own targeted content
                        </p>
                      </div>
                    </a>
                  </div>
                </div>
              </Popover.Panel>
            </Transition>
          </>
        )}
      </Popover>
    </div>
  );
}

export default MyPopover;
```

### RadioGroup

```tsx
import { RadioGroup } from '@headlessui/react';
import { useState } from 'react';

const plans = [
  { name: 'Startup', ram: '12GB', cpus: '6 CPUs', disk: '160 GB SSD disk' },
  { name: 'Business', ram: '16GB', cpus: '8 CPUs', disk: '512 GB SSD disk' },
  { name: 'Enterprise', ram: '32GB', cpus: '12 CPUs', disk: '1024 GB SSD disk' },
];

function MyRadioGroup() {
  const [selected, setSelected] = useState(plans[0]);

  return (
    <div className="w-full px-4 py-16">
      <div className="mx-auto w-full max-w-md">
        <RadioGroup value={selected} onChange={setSelected}>
          <RadioGroup.Label className="sr-only">Server size</RadioGroup.Label>
          <div className="space-y-2">
            {plans.map((plan) => (
              <RadioGroup.Option
                key={plan.name}
                value={plan}
                className={({ active, checked }) =>
                  `${active ? 'ring-2 ring-white ring-opacity-60 ring-offset-2 ring-offset-sky-300' : ''}
                  ${checked ? 'bg-sky-900 bg-opacity-75 text-white' : 'bg-white'}
                    relative flex cursor-pointer rounded-lg px-5 py-4 shadow-md focus:outline-none`
                }
              >
                {({ active, checked }) => (
                  <>
                    <div className="flex w-full items-center justify-between">
                      <div className="flex items-center">
                        <div className="text-sm">
                          <RadioGroup.Label
                            as="p"
                            className={`font-medium  ${
                              checked ? 'text-white' : 'text-gray-900'
                            }`}
                          >
                            {plan.name}
                          </RadioGroup.Label>
                          <RadioGroup.Description
                            as="span"
                            className={`inline ${
                              checked ? 'text-sky-100' : 'text-gray-500'
                            }`}
                          >
                            <span>
                              {plan.ram}/{plan.cpus}
                            </span>{' '}
                            <span aria-hidden="true">&middot;</span>{' '}
                            <span>{plan.disk}</span>
                          </RadioGroup.Description>
                        </div>
                      </div>
                      {checked && (
                        <div className="shrink-0 text-white">
                          <CheckIcon className="h-6 w-6" />
                        </div>
                      )}
                    </div>
                  </>
                )}
              </RadioGroup.Option>
            ))}
          </div>
        </RadioGroup>
      </div>
    </div>
  );
}

function CheckIcon(props: React.SVGProps<SVGSVGElement>) {
  return (
    <svg viewBox="0 0 24 24" fill="none" {...props}>
      <circle cx={12} cy={12} r={12} fill="#fff" opacity="0.2" />
      <path
        d="M7 13l3 3 7-7"
        stroke="#fff"
        strokeWidth={1.5}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
}

export default MyRadioGroup;
```

## Advanced Patterns

### Custom Hook for Modal State

```tsx
import { useState, useCallback } from 'react';

export function useModal(initialState = false) {
  const [isOpen, setIsOpen] = useState(initialState);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen((prev) => !prev), []);

  return { isOpen, open, close, toggle };
}

// Usage
function MyComponent() {
  const modal = useModal();

  return (
    <>
      <button onClick={modal.open}>Open Modal</button>
      <Dialog open={modal.isOpen} onClose={modal.close}>
        {/* Dialog content */}
      </Dialog>
    </>
  );
}
```

### Composing Multiple Dropdowns

```tsx
import { Menu } from '@headlessui/react';

function NestedMenus() {
  return (
    <div className="flex gap-4">
      <Menu>
        <Menu.Button>File</Menu.Button>
        <Menu.Items>
          <Menu.Item>
            {({ active }) => (
              <a className={active ? 'bg-blue-500' : ''} href="/new">
                New
              </a>
            )}
          </Menu.Item>
          <Menu.Item>
            {({ active }) => (
              <a className={active ? 'bg-blue-500' : ''} href="/open">
                Open
              </a>
            )}
          </Menu.Item>
        </Menu.Items>
      </Menu>

      <Menu>
        <Menu.Button>Edit</Menu.Button>
        <Menu.Items>
          <Menu.Item>
            {({ active }) => (
              <button className={active ? 'bg-blue-500' : ''}>Undo</button>
            )}
          </Menu.Item>
          <Menu.Item>
            {({ active }) => (
              <button className={active ? 'bg-blue-500' : ''}>Redo</button>
            )}
          </Menu.Item>
        </Menu.Items>
      </Menu>
    </div>
  );
}
```

## Common Mistakes

1. **Not using Fragment for Transition**: Required to avoid extra DOM nodes
```tsx
// Bad
<Transition enter="..." leave="...">

// Good
<Transition as={Fragment} enter="..." leave="...">
```

2. **Forgetting render props**: Many components provide state via render props
```tsx
// Bad
<Menu.Item>
  <a href="#">Item</a>
</Menu.Item>

// Good
<Menu.Item>
  {({ active }) => (
    <a className={active ? 'bg-blue-500' : ''} href="#">Item</a>
  )}
</Menu.Item>
```

3. **Improper Dialog structure**: Dialog requires specific hierarchy
```tsx
// Bad
<Dialog>
  <Dialog.Panel />
</Dialog>

// Good
<Dialog>
  <Dialog.Panel>
    <Dialog.Title />
    {/* content */}
  </Dialog.Panel>
</Dialog>
```

4. **Not handling keyboard navigation**: Always test with Tab and Arrow keys
```tsx
// Ensure focusable elements and proper ARIA
<Menu.Button>Options</Menu.Button>
```

5. **Missing accessibility labels**: Screen reader support requires labels
```tsx
// Bad
<Switch checked={enabled} onChange={setEnabled} />

// Good
<Switch checked={enabled} onChange={setEnabled}>
  <span className="sr-only">Enable notifications</span>
</Switch>
```

## Best Practices

1. **Use with Tailwind CSS**: Designed specifically for Tailwind integration
2. **Leverage Transition component**: Smooth animations enhance UX
3. **Test accessibility**: Use keyboard and screen readers
4. **Use render props pattern**: Access component state for styling
5. **Combine with Heroicons**: Official icon library from Tailwind Labs
6. **Keep markup semantic**: Use proper HTML elements
7. **Handle focus management**: Components handle it automatically
8. **Use TypeScript**: Excellent type definitions provided
9. **Implement loading states**: Show feedback during async operations
10. **Follow Tailwind conventions**: Use utility classes consistently

## When to Use Headless UI

### Use Headless UI When:
- Working with Tailwind CSS
- Want unstyled, accessible components
- Need full styling control
- Building custom design systems
- Prefer utility-first CSS approach
- Want minimal JavaScript overhead
- Need excellent accessibility out of the box
- Building Tailwind-based projects

### Consider Alternatives When:
- Not using Tailwind CSS (consider Radix UI)
- Need pre-styled components (use Chakra or Mantine)
- Want more component variety (Radix has more primitives)
- Building non-Tailwind projects
- Need complex data table components (use Ant Design)

## Interview Questions

1. **Q: What makes Headless UI different from other component libraries?**
   A: Headless UI is specifically designed for Tailwind CSS, providing unstyled but fully accessible components. It's built by Tailwind Labs, ensuring perfect integration with Tailwind's utility-first approach.

2. **Q: How does Headless UI handle accessibility?**
   A: It implements WAI-ARIA patterns, manages focus automatically, provides keyboard navigation, includes proper ARIA attributes, and supports screen readers. All components follow accessibility guidelines.

3. **Q: Explain the render props pattern in Headless UI.**
   A: Components provide state (like active, selected, open) via render props, allowing conditional styling based on component state. Example: `{({ active }) => <a className={active ? 'bg-blue' : ''} />}`

4. **Q: What is the Transition component used for?**
   A: Transition animates components entering/leaving the DOM with CSS classes. It provides enter/leave lifecycle hooks for smooth animations when showing/hiding content.

5. **Q: How do you integrate Headless UI with Tailwind CSS?**
   A: Use Tailwind utility classes directly on Headless UI components. Leverage render props for conditional classes, use arbitrary variants for state-based styling, and combine with Tailwind's responsive and hover utilities.

## Key Takeaways

1. Headless UI is purpose-built for Tailwind CSS integration
2. Components are completely unstyled, providing maximum flexibility
3. Accessibility is built-in with WAI-ARIA patterns
4. Render props pattern enables state-based styling
5. Transition component provides smooth animations
6. Perfect for utility-first CSS workflows
7. Smaller API surface than Radix UI but tighter Tailwind integration
8. Excellent TypeScript support throughout
9. Focus management and keyboard navigation handled automatically
10. Best choice for Tailwind-based design systems

## Resources

- **Official Documentation**: https://headlessui.com/
- **GitHub Repository**: https://github.com/tailwindlabs/headlessui
- **Tailwind CSS**: https://tailwindcss.com/
- **Heroicons**: https://heroicons.com/
- **Tailwind UI**: https://tailwindui.com/ (Premium components)
- **Component Examples**: https://headlessui.com/react/menu
- **Accessibility Guidelines**: https://www.w3.org/WAI/ARIA/apg/
- **Discord Community**: https://discord.gg/tailwindcss
- **YouTube Tutorials**: Tailwind Labs official channel
- **NPM Package**: https://www.npmjs.com/package/@headlessui/react
