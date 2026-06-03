# Ant Design (antd)

## Introduction

Ant Design is an enterprise-class UI design language and React component library developed by Alibaba. It provides a comprehensive set of high-quality components and design resources for building rich, interactive user interfaces with excellent design consistency and user experience.

## Installation and Setup

### Basic Installation

```bash
npm install antd

# or with yarn
yarn add antd

# or with pnpm
pnpm add antd
```

### App Router Setup (Next.js 13+)

```tsx
// app/layout.tsx
import { AntdRegistry } from '@ant-design/nextjs-registry';
import './globals.css';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <AntdRegistry>{children}</AntdRegistry>
      </body>
    </html>
  );
}
```

### Pages Router Setup (Next.js)

```tsx
// pages/_app.tsx
import { ConfigProvider } from 'antd';
import type { AppProps } from 'next/app';
import 'antd/dist/reset.css';

export default function App({ Component, pageProps }: AppProps) {
  return (
    <ConfigProvider>
      <Component {...pageProps} />
    </ConfigProvider>
  );
}
```

### Custom Theme Configuration

```tsx
// theme/themeConfig.ts
import type { ThemeConfig } from 'antd';

const theme: ThemeConfig = {
  token: {
    colorPrimary: '#1890ff',
    colorSuccess: '#52c41a',
    colorWarning: '#faad14',
    colorError: '#ff4d4f',
    colorInfo: '#1890ff',
    colorTextBase: '#000000',
    colorBgBase: '#ffffff',
    fontSize: 14,
    borderRadius: 6,
    wireframe: false,
  },
  components: {
    Button: {
      colorPrimary: '#00b96b',
      algorithm: true, // Enable algorithm
    },
    Input: {
      colorPrimary: '#eb2f96',
    },
  },
};

export default theme;

// app/layout.tsx
import { ConfigProvider } from 'antd';
import theme from './theme/themeConfig';

export default function RootLayout({ children }) {
  return (
    <ConfigProvider theme={theme}>
      {children}
    </ConfigProvider>
  );
}
```

## Core Components

### Form Components

```tsx
import {
  Form,
  Input,
  Button,
  Select,
  DatePicker,
  InputNumber,
  Checkbox,
  Radio,
  Switch,
  message,
} from 'antd';
import { UserOutlined, LockOutlined, MailOutlined } from '@ant-design/icons';

const { Option } = Select;
const { TextArea } = Input;

function RegistrationForm() {
  const [form] = Form.useForm();

  const onFinish = (values: any) => {
    console.log('Form values:', values);
    message.success('Registration successful!');
  };

  const onFinishFailed = (errorInfo: any) => {
    console.log('Failed:', errorInfo);
    message.error('Please check the form for errors');
  };

  return (
    <Form
      form={form}
      name="registration"
      layout="vertical"
      onFinish={onFinish}
      onFinishFailed={onFinishFailed}
      autoComplete="off"
      initialValues={{
        prefix: '+1',
        newsletter: true,
      }}
    >
      <Form.Item
        label="Username"
        name="username"
        rules={[
          { required: true, message: 'Please input your username!' },
          { min: 3, message: 'Username must be at least 3 characters' },
        ]}
      >
        <Input
          prefix={<UserOutlined />}
          placeholder="Enter username"
          size="large"
        />
      </Form.Item>

      <Form.Item
        label="Email"
        name="email"
        rules={[
          { required: true, message: 'Please input your email!' },
          { type: 'email', message: 'Please enter a valid email!' },
        ]}
      >
        <Input
          prefix={<MailOutlined />}
          placeholder="Enter email"
          size="large"
        />
      </Form.Item>

      <Form.Item
        label="Password"
        name="password"
        rules={[
          { required: true, message: 'Please input your password!' },
          { min: 8, message: 'Password must be at least 8 characters' },
        ]}
      >
        <Input.Password
          prefix={<LockOutlined />}
          placeholder="Enter password"
          size="large"
        />
      </Form.Item>

      <Form.Item
        label="Confirm Password"
        name="confirmPassword"
        dependencies={['password']}
        rules={[
          { required: true, message: 'Please confirm your password!' },
          ({ getFieldValue }) => ({
            validator(_, value) {
              if (!value || getFieldValue('password') === value) {
                return Promise.resolve();
              }
              return Promise.reject(new Error('Passwords do not match!'));
            },
          }),
        ]}
      >
        <Input.Password
          prefix={<LockOutlined />}
          placeholder="Confirm password"
          size="large"
        />
      </Form.Item>

      <Form.Item
        label="Age"
        name="age"
        rules={[{ required: true, message: 'Please input your age!' }]}
      >
        <InputNumber
          min={18}
          max={120}
          style={{ width: '100%' }}
          size="large"
        />
      </Form.Item>

      <Form.Item
        label="Country"
        name="country"
        rules={[{ required: true, message: 'Please select your country!' }]}
      >
        <Select placeholder="Select a country" size="large">
          <Option value="usa">United States</Option>
          <Option value="uk">United Kingdom</Option>
          <Option value="canada">Canada</Option>
          <Option value="australia">Australia</Option>
        </Select>
      </Form.Item>

      <Form.Item
        label="Birth Date"
        name="birthDate"
        rules={[{ required: true, message: 'Please select your birth date!' }]}
      >
        <DatePicker style={{ width: '100%' }} size="large" />
      </Form.Item>

      <Form.Item label="Gender" name="gender" rules={[{ required: true }]}>
        <Radio.Group>
          <Radio value="male">Male</Radio>
          <Radio value="female">Female</Radio>
          <Radio value="other">Other</Radio>
        </Radio.Group>
      </Form.Item>

      <Form.Item label="Bio" name="bio">
        <TextArea rows={4} placeholder="Tell us about yourself" />
      </Form.Item>

      <Form.Item name="newsletter" valuePropName="checked">
        <Checkbox>Subscribe to newsletter</Checkbox>
      </Form.Item>

      <Form.Item name="terms" valuePropName="checked" rules={[
        {
          validator: (_, value) =>
            value ? Promise.resolve() : Promise.reject(new Error('Please accept terms')),
        },
      ]}>
        <Checkbox>I agree to the terms and conditions</Checkbox>
      </Form.Item>

      <Form.Item>
        <Button type="primary" htmlType="submit" size="large" block>
          Register
        </Button>
      </Form.Item>
    </Form>
  );
}

export default RegistrationForm;
```

### Data Display - Table

```tsx
import { Table, Tag, Space, Button, Input, message } from 'antd';
import {
  SearchOutlined,
  EditOutlined,
  DeleteOutlined,
  EyeOutlined,
} from '@ant-design/icons';
import type { ColumnsType, TablePaginationConfig } from 'antd/es/table';
import { useState } from 'react';

interface DataType {
  key: string;
  name: string;
  age: number;
  address: string;
  tags: string[];
  email: string;
  status: 'active' | 'inactive' | 'pending';
}

function DataTable() {
  const [loading, setLoading] = useState(false);
  const [selectedRowKeys, setSelectedRowKeys] = useState<React.Key[]>([]);

  const data: DataType[] = [
    {
      key: '1',
      name: 'John Brown',
      age: 32,
      address: 'New York No. 1 Lake Park',
      tags: ['developer', 'senior'],
      email: 'john@example.com',
      status: 'active',
    },
    {
      key: '2',
      name: 'Jim Green',
      age: 42,
      address: 'London No. 1 Lake Park',
      tags: ['manager'],
      email: 'jim@example.com',
      status: 'inactive',
    },
    {
      key: '3',
      name: 'Joe Black',
      age: 32,
      address: 'Sydney No. 1 Lake Park',
      tags: ['designer', 'junior'],
      email: 'joe@example.com',
      status: 'pending',
    },
  ];

  const columns: ColumnsType<DataType> = [
    {
      title: 'Name',
      dataIndex: 'name',
      key: 'name',
      sorter: (a, b) => a.name.localeCompare(b.name),
      filterDropdown: ({ setSelectedKeys, selectedKeys, confirm, clearFilters }) => (
        <div style={{ padding: 8 }}>
          <Input
            placeholder="Search name"
            value={selectedKeys[0]}
            onChange={(e) => setSelectedKeys(e.target.value ? [e.target.value] : [])}
            onPressEnter={() => confirm()}
            style={{ marginBottom: 8, display: 'block' }}
          />
          <Space>
            <Button
              type="primary"
              onClick={() => confirm()}
              icon={<SearchOutlined />}
              size="small"
            >
              Search
            </Button>
            <Button onClick={clearFilters} size="small">
              Reset
            </Button>
          </Space>
        </div>
      ),
      filterIcon: (filtered) => (
        <SearchOutlined style={{ color: filtered ? '#1890ff' : undefined }} />
      ),
      onFilter: (value, record) =>
        record.name.toLowerCase().includes(value.toString().toLowerCase()),
    },
    {
      title: 'Age',
      dataIndex: 'age',
      key: 'age',
      sorter: (a, b) => a.age - b.age,
      width: 100,
    },
    {
      title: 'Email',
      dataIndex: 'email',
      key: 'email',
    },
    {
      title: 'Address',
      dataIndex: 'address',
      key: 'address',
    },
    {
      title: 'Status',
      key: 'status',
      dataIndex: 'status',
      render: (status) => {
        const colorMap = {
          active: 'green',
          inactive: 'red',
          pending: 'orange',
        };
        return <Tag color={colorMap[status]}>{status.toUpperCase()}</Tag>;
      },
      filters: [
        { text: 'Active', value: 'active' },
        { text: 'Inactive', value: 'inactive' },
        { text: 'Pending', value: 'pending' },
      ],
      onFilter: (value, record) => record.status === value,
    },
    {
      title: 'Tags',
      key: 'tags',
      dataIndex: 'tags',
      render: (tags: string[]) => (
        <>
          {tags.map((tag) => (
            <Tag color="blue" key={tag}>
              {tag}
            </Tag>
          ))}
        </>
      ),
    },
    {
      title: 'Action',
      key: 'action',
      render: (_, record) => (
        <Space size="middle">
          <Button
            type="link"
            icon={<EyeOutlined />}
            onClick={() => message.info(`View ${record.name}`)}
          >
            View
          </Button>
          <Button
            type="link"
            icon={<EditOutlined />}
            onClick={() => message.info(`Edit ${record.name}`)}
          >
            Edit
          </Button>
          <Button
            type="link"
            danger
            icon={<DeleteOutlined />}
            onClick={() => message.warning(`Delete ${record.name}`)}
          >
            Delete
          </Button>
        </Space>
      ),
    },
  ];

  const rowSelection = {
    selectedRowKeys,
    onChange: (selectedKeys: React.Key[]) => {
      setSelectedRowKeys(selectedKeys);
      console.log('Selected Keys:', selectedKeys);
    },
  };

  const handleTableChange = (
    pagination: TablePaginationConfig,
    filters: any,
    sorter: any
  ) => {
    console.log('Table params:', { pagination, filters, sorter });
  };

  return (
    <div>
      <div style={{ marginBottom: 16 }}>
        <Button
          type="primary"
          disabled={selectedRowKeys.length === 0}
          loading={loading}
        >
          Bulk Action ({selectedRowKeys.length} selected)
        </Button>
      </div>
      <Table
        columns={columns}
        dataSource={data}
        rowSelection={rowSelection}
        loading={loading}
        onChange={handleTableChange}
        pagination={{
          pageSize: 10,
          showSizeChanger: true,
          showTotal: (total) => `Total ${total} items`,
        }}
      />
    </div>
  );
}

export default DataTable;
```

### Modal and Drawer

```tsx
import { useState } from 'react';
import { Modal, Drawer, Button, Space, Form, Input, message } from 'antd';
import { ExclamationCircleOutlined } from '@ant-design/icons';

const { confirm } = Modal;

function ModalDrawerExample() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isDrawerOpen, setIsDrawerOpen] = useState(false);
  const [form] = Form.useForm();

  const showModal = () => {
    setIsModalOpen(true);
  };

  const handleModalOk = () => {
    form.validateFields().then((values) => {
      console.log('Form values:', values);
      message.success('Form submitted successfully');
      setIsModalOpen(false);
      form.resetFields();
    });
  };

  const handleModalCancel = () => {
    setIsModalOpen(false);
    form.resetFields();
  };

  const showDrawer = () => {
    setIsDrawerOpen(true);
  };

  const closeDrawer = () => {
    setIsDrawerOpen(false);
  };

  const showConfirm = () => {
    confirm({
      title: 'Do you want to delete these items?',
      icon: <ExclamationCircleOutlined />,
      content: 'This action cannot be undone.',
      onOk() {
        return new Promise((resolve) => {
          setTimeout(() => {
            message.success('Items deleted');
            resolve(null);
          }, 1000);
        });
      },
      onCancel() {
        console.log('Cancel');
      },
    });
  };

  const showPromiseConfirm = () => {
    Modal.confirm({
      title: 'Async Confirmation',
      content: 'This will perform an async operation',
      async onOk() {
        await new Promise((resolve) => setTimeout(resolve, 2000));
        message.success('Async operation completed');
      },
    });
  };

  return (
    <Space direction="vertical" style={{ width: '100%' }}>
      <Space>
        <Button type="primary" onClick={showModal}>
          Open Modal
        </Button>
        <Button onClick={showDrawer}>Open Drawer</Button>
        <Button onClick={showConfirm} danger>
          Show Confirm
        </Button>
        <Button onClick={showPromiseConfirm}>Async Confirm</Button>
      </Space>

      <Modal
        title="Create New User"
        open={isModalOpen}
        onOk={handleModalOk}
        onCancel={handleModalCancel}
        width={600}
        okText="Submit"
        cancelText="Cancel"
      >
        <Form
          form={form}
          layout="vertical"
          style={{ marginTop: 20 }}
        >
          <Form.Item
            label="Username"
            name="username"
            rules={[{ required: true, message: 'Please input username!' }]}
          >
            <Input placeholder="Enter username" />
          </Form.Item>
          <Form.Item
            label="Email"
            name="email"
            rules={[
              { required: true, message: 'Please input email!' },
              { type: 'email', message: 'Invalid email format!' },
            ]}
          >
            <Input placeholder="Enter email" />
          </Form.Item>
        </Form>
      </Modal>

      <Drawer
        title="User Settings"
        placement="right"
        onClose={closeDrawer}
        open={isDrawerOpen}
        width={400}
      >
        <Form layout="vertical">
          <Form.Item label="Display Name">
            <Input placeholder="Enter display name" />
          </Form.Item>
          <Form.Item label="Bio">
            <Input.TextArea rows={4} placeholder="Tell us about yourself" />
          </Form.Item>
          <Form.Item>
            <Space>
              <Button type="primary">Save</Button>
              <Button onClick={closeDrawer}>Cancel</Button>
            </Space>
          </Form.Item>
        </Form>
      </Drawer>
    </Space>
  );
}

export default ModalDrawerExample;
```

### Navigation - Menu

```tsx
import { Menu, Layout } from 'antd';
import {
  MailOutlined,
  AppstoreOutlined,
  SettingOutlined,
  UserOutlined,
  TeamOutlined,
} from '@ant-design/icons';
import type { MenuProps } from 'antd';
import { useState } from 'react';

const { Sider } = Layout;

type MenuItem = Required<MenuProps>['items'][number];

function getItem(
  label: React.ReactNode,
  key: React.Key,
  icon?: React.ReactNode,
  children?: MenuItem[],
  type?: 'group'
): MenuItem {
  return {
    key,
    icon,
    children,
    label,
    type,
  } as MenuItem;
}

const items: MenuItem[] = [
  getItem('Dashboard', 'dashboard', <AppstoreOutlined />),
  getItem('Users', 'users', <UserOutlined />, [
    getItem('All Users', 'all-users'),
    getItem('Add User', 'add-user'),
    getItem('User Roles', 'user-roles'),
  ]),
  getItem('Teams', 'teams', <TeamOutlined />, [
    getItem('Team 1', 'team1'),
    getItem('Team 2', 'team2'),
  ]),
  getItem('Messages', 'messages', <MailOutlined />, [
    getItem('Inbox', 'inbox'),
    getItem('Sent', 'sent'),
    getItem('Drafts', 'drafts'),
  ]),
  { type: 'divider' },
  getItem('Settings', 'settings', <SettingOutlined />, [
    getItem('Profile', 'profile'),
    getItem('Security', 'security'),
    getItem('Notifications', 'notifications'),
  ]),
];

function NavigationMenu() {
  const [collapsed, setCollapsed] = useState(false);
  const [selectedKey, setSelectedKey] = useState('dashboard');

  const onClick: MenuProps['onClick'] = (e) => {
    console.log('Selected menu item:', e.key);
    setSelectedKey(e.key);
  };

  return (
    <Layout style={{ minHeight: '100vh' }}>
      <Sider
        collapsible
        collapsed={collapsed}
        onCollapse={setCollapsed}
        theme="light"
      >
        <div
          style={{
            height: 32,
            margin: 16,
            background: 'rgba(0, 0, 0, 0.2)',
            borderRadius: 6,
          }}
        />
        <Menu
          onClick={onClick}
          selectedKeys={[selectedKey]}
          defaultOpenKeys={['users', 'messages']}
          mode="inline"
          items={items}
        />
      </Sider>
      <Layout.Content style={{ padding: 24 }}>
        <h2>Selected: {selectedKey}</h2>
      </Layout.Content>
    </Layout>
  );
}

export default NavigationMenu;
```

### Feedback - Notifications and Messages

```tsx
import { Button, Space, notification, message, Modal } from 'antd';
import {
  SmileOutlined,
  CheckCircleOutlined,
  CloseCircleOutlined,
  InfoCircleOutlined,
  WarningOutlined,
} from '@ant-design/icons';

function FeedbackComponents() {
  const [api, contextHolder] = notification.useNotification();
  const [modal, modalContextHolder] = Modal.useModal();

  const openNotification = (type: 'success' | 'info' | 'warning' | 'error') => {
    api[type]({
      message: `${type.charAt(0).toUpperCase() + type.slice(1)} Notification`,
      description:
        'This is the content of the notification. It provides detailed information.',
      duration: 4.5,
      placement: 'topRight',
    });
  };

  const openCustomNotification = () => {
    api.open({
      message: 'Custom Notification',
      description: 'This notification has a custom icon and duration.',
      icon: <SmileOutlined style={{ color: '#108ee9' }} />,
      duration: 0, // Won't auto close
      btn: (
        <Button type="primary" size="small" onClick={() => api.destroy()}>
          Close All
        </Button>
      ),
    });
  };

  const showMessage = (type: 'success' | 'error' | 'warning' | 'info') => {
    message[type](`This is a ${type} message`);
  };

  const showLoadingMessage = () => {
    const hide = message.loading('Action in progress..', 0);
    setTimeout(() => {
      hide();
      message.success('Action completed!');
    }, 2000);
  };

  const showModalConfirm = () => {
    modal.confirm({
      title: 'Confirm Action',
      icon: <ExclamationCircleOutlined />,
      content: 'Are you sure you want to proceed?',
      onOk: () => {
        return new Promise((resolve) => {
          setTimeout(() => {
            message.success('Action confirmed');
            resolve(null);
          }, 1000);
        });
      },
    });
  };

  return (
    <>
      {contextHolder}
      {modalContextHolder}
      <Space direction="vertical" style={{ width: '100%' }}>
        <Space wrap>
          <Button onClick={() => openNotification('success')}>
            Success Notification
          </Button>
          <Button onClick={() => openNotification('info')}>
            Info Notification
          </Button>
          <Button onClick={() => openNotification('warning')}>
            Warning Notification
          </Button>
          <Button onClick={() => openNotification('error')}>
            Error Notification
          </Button>
          <Button onClick={openCustomNotification}>Custom Notification</Button>
        </Space>

        <Space wrap>
          <Button onClick={() => showMessage('success')}>Success Message</Button>
          <Button onClick={() => showMessage('error')}>Error Message</Button>
          <Button onClick={() => showMessage('warning')}>Warning Message</Button>
          <Button onClick={() => showMessage('info')}>Info Message</Button>
          <Button onClick={showLoadingMessage}>Loading Message</Button>
        </Space>

        <Button onClick={showModalConfirm}>Show Confirm Modal</Button>
      </Space>
    </>
  );
}

export default FeedbackComponents;
```

## Advanced Features

### Custom Theme with Design Tokens

```tsx
// theme/customTheme.ts
import type { ThemeConfig } from 'antd';

export const customTheme: ThemeConfig = {
  token: {
    // Primary colors
    colorPrimary: '#722ed1',
    colorSuccess: '#52c41a',
    colorWarning: '#faad14',
    colorError: '#f5222d',
    colorInfo: '#1890ff',
    
    // Text colors
    colorTextBase: '#262626',
    colorTextSecondary: '#8c8c8c',
    colorTextTertiary: '#bfbfbf',
    colorTextQuaternary: '#d9d9d9',
    
    // Background colors
    colorBgBase: '#ffffff',
    colorBgContainer: '#f5f5f5',
    colorBgElevated: '#ffffff',
    colorBgLayout: '#f0f2f5',
    
    // Border
    colorBorder: '#d9d9d9',
    colorBorderSecondary: '#f0f0f0',
    borderRadius: 8,
    
    // Typography
    fontSize: 14,
    fontSizeHeading1: 38,
    fontSizeHeading2: 30,
    fontSizeHeading3: 24,
    fontSizeHeading4: 20,
    fontSizeHeading5: 16,
    
    // Spacing
    marginXS: 8,
    marginSM: 12,
    margin: 16,
    marginMD: 20,
    marginLG: 24,
    marginXL: 32,
    
    // Animation
    motionDurationFast: '0.1s',
    motionDurationMid: '0.2s',
    motionDurationSlow: '0.3s',
    
    // Z-index
    zIndexPopupBase: 1000,
    zIndexBase: 0,
  },
  components: {
    Button: {
      primaryShadow: '0 2px 0 rgba(114, 46, 209, 0.1)',
      controlHeight: 40,
      controlHeightLG: 48,
      controlHeightSM: 32,
    },
    Input: {
      controlHeight: 40,
      controlHeightLG: 48,
      controlHeightSM: 32,
    },
    Select: {
      controlHeight: 40,
    },
    Table: {
      headerBg: '#fafafa',
      headerColor: '#262626',
      rowHoverBg: '#f5f5f5',
    },
    Card: {
      boxShadowTertiary: '0 1px 2px 0 rgba(0, 0, 0, 0.03), 0 1px 6px -1px rgba(0, 0, 0, 0.02), 0 2px 4px 0 rgba(0, 0, 0, 0.02)',
    },
  },
  algorithm: undefined, // Can use theme.darkAlgorithm or theme.compactAlgorithm
};

// Usage in app
import { ConfigProvider, theme } from 'antd';
import { customTheme } from './theme/customTheme';

function App() {
  return (
    <ConfigProvider
      theme={{
        ...customTheme,
        algorithm: theme.defaultAlgorithm, // or theme.darkAlgorithm
      }}
    >
      {/* Your app */}
    </ConfigProvider>
  );
}
```

### Internationalization (i18n)

```tsx
import { ConfigProvider } from 'antd';
import enUS from 'antd/locale/en_US';
import zhCN from 'antd/locale/zh_CN';
import esES from 'antd/locale/es_ES';
import frFR from 'antd/locale/fr_FR';
import { useState } from 'react';

function I18nExample() {
  const [locale, setLocale] = useState(enUS);

  const locales = {
    'en-US': enUS,
    'zh-CN': zhCN,
    'es-ES': esES,
    'fr-FR': frFR,
  };

  return (
    <ConfigProvider locale={locale}>
      <Space>
        {Object.keys(locales).map((key) => (
          <Button
            key={key}
            onClick={() => setLocale(locales[key as keyof typeof locales])}
          >
            {key}
          </Button>
        ))}
      </Space>
      {/* Your components will use the selected locale */}
    </ConfigProvider>
  );
}
```

### Custom Hooks

```tsx
import { Form, message } from 'antd';
import type { FormInstance } from 'antd';

// Custom form hook with validation
export function useFormValidation<T = any>() {
  const [form] = Form.useForm<T>();
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (
    onSubmit: (values: T) => Promise<void>,
    onError?: (error: any) => void
  ) => {
    try {
      setLoading(true);
      const values = await form.validateFields();
      await onSubmit(values);
      message.success('Form submitted successfully');
      form.resetFields();
    } catch (error) {
      if (onError) {
        onError(error);
      } else {
        message.error('Form validation failed');
      }
      console.error('Form validation error:', error);
    } finally {
      setLoading(false);
    }
  };

  return {
    form,
    loading,
    handleSubmit,
  };
}

// Usage
function MyForm() {
  const { form, loading, handleSubmit } = useFormValidation();

  const onSubmit = async (values: any) => {
    await api.post('/users', values);
  };

  return (
    <Form form={form} onFinish={() => handleSubmit(onSubmit)}>
      {/* Form fields */}
      <Button type="primary" htmlType="submit" loading={loading}>
        Submit
      </Button>
    </Form>
  );
}
```

## Common Mistakes

1. **Not using Form.useForm()**: Always use the hook for better control
```tsx
// Bad
<Form>

// Good
const [form] = Form.useForm();
<Form form={form}>
```

2. **Missing ConfigProvider**: Required for proper theming and locale
```tsx
// Bad - Direct component usage without provider
<Button />

// Good
<ConfigProvider theme={customTheme}>
  <Button />
</ConfigProvider>
```

3. **Not handling async operations in modals**
```tsx
// Bad
Modal.confirm({
  onOk() {
    api.delete(); // Fire and forget
  }
});

// Good
Modal.confirm({
  async onOk() {
    await api.delete();
    message.success('Deleted');
  }
});
```

4. **Improper table key prop**
```tsx
// Bad - Using index as key
data.map((item, index) => ({ ...item, key: index }))

// Good - Using unique identifier
data.map(item => ({ ...item, key: item.id }))
```

5. **Not cleaning up notification instances**
```tsx
// Bad - Can cause memory leaks
notification.open({ duration: 0 });

// Good - Store instance and destroy
const key = `notification-${Date.now()}`;
notification.open({ key, duration: 0 });
// Later: notification.close(key);
```

## Best Practices

1. **Use TypeScript**: Leverage Ant Design's excellent type definitions
2. **Customize theme globally**: Use ConfigProvider for consistent theming
3. **Lazy load components**: Import only what you need for better performance
4. **Use Form.Item dependencies**: For related field validations
5. **Implement proper error boundaries**: Catch and handle component errors
6. **Use notification hooks**: Better for testing and control
7. **Follow accessibility guidelines**: Ant Design has built-in ARIA support
8. **Optimize table rendering**: Use pagination and virtualization for large datasets
9. **Cache form values**: Use form.getFieldsValue() to preserve state
10. **Use message feedback**: Provide clear user feedback for all actions

## When to Use Ant Design

### Use Ant Design When:
- Building enterprise applications with complex data tables and forms
- Need a comprehensive component library with consistent design
- Want excellent TypeScript support out of the box
- Building admin dashboards, management systems, or data-heavy applications
- Need internationalization support
- Want a mature, well-documented library with large community

### Consider Alternatives When:
- Need minimal, customizable components (use Radix UI or Headless UI)
- Building marketing sites or simple landing pages
- Need a different design language (Material UI for Material Design)
- Want more control over styling (Chakra UI or Tailwind components)
- Bundle size is critical concern (Ant Design is feature-rich but larger)

## Interview Questions

1. **Q: How does Ant Design's theming system work?**
   A: Ant Design uses design tokens in the ConfigProvider with algorithm support for dark/compact themes. Tokens cascade through components using CSS-in-JS and can be customized at global or component level.

2. **Q: Explain Form validation in Ant Design.**
   A: Form validation uses async-validator library with rules prop. It supports required, type, custom validators, and dependencies. Form.useForm() provides methods like validateFields, setFieldsValue, and getFieldValue.

3. **Q: What is the difference between message and notification?**
   A: message provides simple, short-lived feedback at top of screen. notification is more prominent with title, description, icon, and custom placement options for important updates.

4. **Q: How do you optimize Table performance with large datasets?**
   A: Use pagination, enable virtual scrolling with rc-virtual-list, implement server-side filtering/sorting, use rowKey properly, and consider React.memo for custom cell renders.

5. **Q: Explain ConfigProvider's role in Ant Design.**
   A: ConfigProvider is the root wrapper providing theme, locale, component config, and global settings via React context. It enables consistent theming, i18n, and component customization across the app.

## Key Takeaways

1. Ant Design is ideal for enterprise applications with complex UIs
2. The theming system is powerful and flexible with design tokens
3. Form handling is robust with comprehensive validation support
4. Components follow accessibility best practices by default
5. TypeScript support is excellent throughout the library
6. The Table component is feature-rich for complex data display
7. Internationalization is built-in with locale support for 30+ languages
8. ConfigProvider is essential for global customization
9. The library balances features with bundle size reasonably well
10. Large community and ecosystem provide extensive third-party components

## Resources

- **Official Documentation**: https://ant.design/docs/react/introduce
- **GitHub Repository**: https://github.com/ant-design/ant-design
- **Component Demos**: https://ant.design/components/overview/
- **Design Resources**: https://ant.design/docs/resources
- **Ant Design Pro**: https://pro.ant.design/ (Enterprise templates)
- **Ant Design Charts**: https://charts.ant.design/
- **Ant Design Mobile**: https://mobile.ant.design/
- **Ant Design Icons**: https://ant.design/components/icon/
- **Figma Toolkit**: https://www.figma.com/community/file/831698976089873405
- **Discord Community**: https://discord.gg/jmNvw4BFhn
