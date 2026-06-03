# Webpack Configuration

## Overview

Webpack is a powerful module bundler for JavaScript applications. It takes modules with dependencies and generates static assets representing those modules. Webpack has become the de facto standard for bundling modern web applications, offering extensive configuration options for optimization, code splitting, and asset management.

## Core Concepts

### Webpack Build Process

```
┌──────────────┐
│ Entry Points │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Loaders    │ Transform files
├──────────────┤ (TypeScript → JS,
│ SCSS → CSS   │  SCSS → CSS, etc.)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Dependency   │ Build dependency
│    Graph     │ graph
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Plugins    │ Additional
├──────────────┤ processing
│ Optimization │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Output    │ Bundled files
│   (dist/)    │
└──────────────┘
```

### Key Concepts

1. **Entry**: Starting point for dependency graph
2. **Output**: Where to emit bundles
3. **Loaders**: Transform files (TypeScript, SCSS, etc.)
4. **Plugins**: Additional processing and optimization
5. **Mode**: Development or production configuration
6. **Code Splitting**: Split code into multiple bundles
7. **Tree Shaking**: Remove unused code

## Basic Configuration

### Minimal Webpack Config

```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  // Entry point
  entry: './src/index.js',
  
  // Output configuration
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
    clean: true // Clean dist folder before each build
  },
  
  // Mode: 'development', 'production', or 'none'
  mode: 'development',
  
  // Development server
  devServer: {
    static: './dist',
    port: 3000,
    hot: true, // Hot Module Replacement
    open: true // Open browser automatically
  }
};
```

### Multiple Entry Points

```javascript
// webpack.config.js
module.exports = {
  entry: {
    app: './src/index.js',
    admin: './src/admin.js',
    vendor: './src/vendor.js'
  },
  
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js', // app.abc123.js, admin.def456.js
    chunkFilename: '[name].[contenthash].chunk.js'
  }
};
```

## Loaders Configuration

### TypeScript Loader

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  },
  
  resolve: {
    extensions: ['.tsx', '.ts', '.js']
  }
};

// Alternative with Babel
module.exports = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: [
                '@babel/preset-env',
                '@babel/preset-typescript',
                '@babel/preset-react'
              ]
            }
          }
        ],
        exclude: /node_modules/
      }
    ]
  }
};
```

### CSS and SCSS Loaders

```javascript
// webpack.config.js
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  module: {
    rules: [
      // CSS
      {
        test: /\.css$/,
        use: [
          process.env.NODE_ENV === 'production'
            ? MiniCssExtractPlugin.loader
            : 'style-loader',
          'css-loader',
          'postcss-loader'
        ]
      },
      
      // SCSS/SASS
      {
        test: /\.s[ac]ss$/,
        use: [
          process.env.NODE_ENV === 'production'
            ? MiniCssExtractPlugin.loader
            : 'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: true, // Enable CSS modules for .module.scss files
                localIdentName: '[name]__[local]--[hash:base64:5]'
              },
              sourceMap: true
            }
          },
          {
            loader: 'postcss-loader',
            options: {
              postcssOptions: {
                plugins: [
                  'autoprefixer',
                  'postcss-preset-env'
                ]
              }
            }
          },
          {
            loader: 'sass-loader',
            options: {
              sourceMap: true
            }
          }
        ]
      }
    ]
  },
  
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
      chunkFilename: '[id].[contenthash].css'
    })
  ]
};
```

### Asset Loaders

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      // Images
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024 // Inline images < 8kb as base64
          }
        },
        generator: {
          filename: 'images/[name].[hash][ext]'
        }
      },
      
      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[name].[hash][ext]'
        }
      },
      
      // SVG as React components
      {
        test: /\.svg$/,
        issuer: /\.[jt]sx?$/,
        use: ['@svgr/webpack']
      }
    ]
  }
};
```

### Babel Loader

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', {
                targets: {
                  browsers: ['last 2 versions', '> 1%', 'not dead']
                },
                useBuiltIns: 'usage',
                corejs: 3
              }],
              '@babel/preset-react'
            ],
            plugins: [
              '@babel/plugin-proposal-class-properties',
              '@babel/plugin-proposal-optional-chaining',
              '@babel/plugin-proposal-nullish-coalescing-operator',
              [
                'babel-plugin-styled-components',
                {
                  displayName: true,
                  fileName: true
                }
              ]
            ],
            cacheDirectory: true // Cache babel compilation
          }
        }
      }
    ]
  }
};

// .babelrc alternative
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.25%, not dead",
      "useBuiltIns": "usage",
      "corejs": 3
    }],
    "@babel/preset-react",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-class-properties",
    "@babel/plugin-proposal-optional-chaining"
  ]
}
```

## Essential Plugins

### HTML Plugin

```javascript
// webpack.config.js
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
      filename: 'index.html',
      inject: 'body',
      scriptLoading: 'defer',
      minify: {
        removeComments: true,
        collapseWhitespace: true,
        removeRedundantAttributes: true,
        useShortDoctype: true,
        removeEmptyAttributes: true,
        removeStyleLinkTypeAttributes: true,
        keepClosingSlash: true,
        minifyJS: true,
        minifyCSS: true,
        minifyURLs: true
      }
    }),
    
    // Multiple HTML pages
    new HtmlWebpackPlugin({
      template: './src/admin.html',
      filename: 'admin.html',
      chunks: ['admin', 'vendor'] // Only include specific chunks
    })
  ]
};
```

### Clean Plugin

```javascript
// webpack.config.js
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = {
  plugins: [
    new CleanWebpackPlugin({
      cleanOnceBeforeBuildPatterns: ['**/*', '!important-file.js'],
      dry: false, // Set to true for dry run
      verbose: true
    })
  ]
};
```

### Copy Plugin

```javascript
// webpack.config.js
const CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = {
  plugins: [
    new CopyWebpackPlugin({
      patterns: [
        {
          from: 'public',
          to: '',
          globOptions: {
            ignore: ['**/index.html']
          }
        },
        {
          from: 'src/assets/images',
          to: 'images'
        }
      ]
    })
  ]
};
```

### Define Plugin

```javascript
// webpack.config.js
const webpack = require('webpack');

module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      'process.env.API_URL': JSON.stringify(process.env.API_URL),
      '__DEV__': process.env.NODE_ENV !== 'production',
      'VERSION': JSON.stringify('1.0.0')
    })
  ]
};
```

### Environment Variables

```javascript
// webpack.config.js
const Dotenv = require('dotenv-webpack');

module.exports = {
  plugins: [
    new Dotenv({
      path: './.env',
      safe: true, // Load .env.example to verify
      systemvars: true, // Load system environment variables
      silent: false,
      defaults: false
    })
  ]
};

// .env file
API_URL=https://api.example.com
API_KEY=secret123
NODE_ENV=development
```

## Code Splitting

### Dynamic Imports

```javascript
// Lazy loading components
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// webpack.config.js
module.exports = {
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js',
    path: path.resolve(__dirname, 'dist')
  },
  
  optimization: {
    splitChunks: {
      chunks: 'all'
    }
  }
};
```

### Split Chunks Configuration

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: Infinity,
      minSize: 0,
      cacheGroups: {
        // Vendor chunk for node_modules
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            // Get package name
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            
            return `npm.${packageName.replace('@', '')}`;
          },
          priority: 10
        },
        
        // Common chunk for shared code
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
          enforce: true
        },
        
        // React libraries
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
          name: 'react-vendors',
          priority: 20
        },
        
        // Utility libraries
        utilities: {
          test: /[\\/]node_modules[\\/](lodash|moment|date-fns)[\\/]/,
          name: 'utilities',
          priority: 15
        }
      }
    },
    
    // Extract runtime into separate chunk
    runtimeChunk: {
      name: 'runtime'
    },
    
    // Use better module IDs for caching
    moduleIds: 'deterministic'
  }
};
```

### Prefetch and Preload

```javascript
// Prefetch - load during idle time
import(/* webpackPrefetch: true */ './lazy-module');

// Preload - load in parallel with parent chunk
import(/* webpackPreload: true */ './critical-module');

// Component example
function MyComponent() {
  const handleClick = () => {
    import(/* webpackChunkName: "heavy-library" */ './heavy-library')
      .then(module => {
        module.doSomething();
      });
  };
  
  return <button onClick={handleClick}>Load Feature</button>;
}
```

## Optimization

### Production Configuration

```javascript
// webpack.prod.js
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = {
  mode: 'production',
  
  devtool: 'source-map',
  
  optimization: {
    minimize: true,
    minimizer: [
      // Minify JavaScript
      new TerserPlugin({
        parallel: true,
        terserOptions: {
          compress: {
            drop_console: true, // Remove console.log
            drop_debugger: true,
            pure_funcs: ['console.log', 'console.info'] // Remove specific functions
          },
          format: {
            comments: false // Remove comments
          },
          mangle: true // Shorten variable names
        },
        extractComments: false
      }),
      
      // Minify CSS
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            'default',
            {
              discardComments: { removeAll: true }
            }
          ]
        }
      })
    ],
    
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        }
      }
    },
    
    runtimeChunk: 'single',
    
    moduleIds: 'deterministic',
    
    usedExports: true, // Tree shaking
    sideEffects: true // Respect package.json sideEffects field
  },
  
  plugins: [
    // Gzip compression
    new CompressionPlugin({
      filename: '[path][base].gz',
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 10240, // Only compress files > 10kb
      minRatio: 0.8
    }),
    
    // Brotli compression
    new CompressionPlugin({
      filename: '[path][base].br',
      algorithm: 'brotliCompress',
      test: /\.(js|css|html|svg)$/,
      compressionOptions: {
        level: 11
      },
      threshold: 10240,
      minRatio: 0.8
    })
  ],
  
  performance: {
    maxEntrypointSize: 512000, // 500kb
    maxAssetSize: 512000,
    hints: 'warning'
  }
};
```

### Bundle Analyzer

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static', // Generates static HTML file
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json'
    })
  ]
};

// package.json
{
  "scripts": {
    "analyze": "webpack --config webpack.prod.js --profile --json > stats.json && webpack-bundle-analyzer stats.json"
  }
}
```

## Development Configuration

### Dev Server

```javascript
// webpack.dev.js
module.exports = {
  mode: 'development',
  
  devtool: 'eval-source-map', // Fast, high-quality source maps
  
  devServer: {
    static: {
      directory: path.join(__dirname, 'public')
    },
    
    port: 3000,
    
    hot: true, // Hot Module Replacement
    
    open: true, // Open browser
    
    compress: true, // Enable gzip
    
    historyApiFallback: true, // SPA routing support
    
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        pathRewrite: { '^/api': '' },
        secure: false,
        changeOrigin: true
      }
    },
    
    headers: {
      'Access-Control-Allow-Origin': '*'
    },
    
    client: {
      overlay: {
        errors: true,
        warnings: false
      },
      progress: true
    }
  },
  
  cache: {
    type: 'filesystem', // Cache to disk for faster rebuilds
    buildDependencies: {
      config: [__filename]
    }
  }
};
```

### Source Maps

```javascript
// Different source map options

// Development
module.exports = {
  devtool: 'eval-source-map' // Best quality, slower
  // or 'eval-cheap-module-source-map' // Faster, good quality
};

// Production
module.exports = {
  devtool: 'source-map' // Separate .map files
  // or 'hidden-source-map' // Source maps without reference
  // or false // No source maps
};
```

## Advanced Configuration

### Module Federation (Micro-frontends)

```javascript
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      
      exposes: {
        './Button': './src/components/Button',
        './Header': './src/components/Header'
      },
      
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js'
      },
      
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0'
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0'
        }
      }
    })
  ]
};
```

### Custom Webpack Plugin

```javascript
// custom-plugin.js
class MyCustomPlugin {
  apply(compiler) {
    compiler.hooks.emit.tapAsync('MyCustomPlugin', (compilation, callback) => {
      console.log('Webpack is emitting assets...');
      
      // Access compilation assets
      const assets = compilation.assets;
      
      // Create custom asset
      compilation.assets['custom-file.txt'] = {
        source: () => 'Custom content',
        size: () => 14
      };
      
      callback();
    });
    
    compiler.hooks.done.tap('MyCustomPlugin', (stats) => {
      console.log('Build completed!');
      console.log('Time:', stats.endTime - stats.startTime, 'ms');
    });
  }
}

module.exports = MyCustomPlugin;

// Usage in webpack.config.js
const MyCustomPlugin = require('./custom-plugin');

module.exports = {
  plugins: [
    new MyCustomPlugin()
  ]
};
```

### Multi-Configuration

```javascript
// webpack.config.js - Export multiple configurations
module.exports = [
  // Client configuration
  {
    name: 'client',
    target: 'web',
    entry: './src/client/index.js',
    output: {
      path: path.resolve(__dirname, 'dist/client'),
      filename: '[name].js'
    }
  },
  
  // Server configuration
  {
    name: 'server',
    target: 'node',
    entry: './src/server/index.js',
    output: {
      path: path.resolve(__dirname, 'dist/server'),
      filename: 'server.js'
    },
    externals: [nodeExternals()]
  }
];

// Run specific configuration
// webpack --config-name client
```

## Complete Real-World Example

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const webpack = require('webpack');

const isDevelopment = process.env.NODE_ENV !== 'production';

module.exports = {
  mode: isDevelopment ? 'development' : 'production',
  
  entry: {
    main: './src/index.tsx'
  },
  
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: isDevelopment
      ? '[name].js'
      : '[name].[contenthash:8].js',
    chunkFilename: isDevelopment
      ? '[name].chunk.js'
      : '[name].[contenthash:8].chunk.js',
    assetModuleFilename: 'assets/[name].[hash][ext]',
    clean: true,
    publicPath: '/'
  },
  
  resolve: {
    extensions: ['.tsx', '.ts', '.jsx', '.js'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@utils': path.resolve(__dirname, 'src/utils')
    }
  },
  
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      },
      {
        test: /\.s[ac]ss$/,
        use: [
          isDevelopment ? 'style-loader' : MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: true,
                localIdentName: isDevelopment
                  ? '[name]__[local]--[hash:base64:5]'
                  : '[hash:base64]'
              }
            }
          },
          'postcss-loader',
          'sass-loader'
        ]
      },
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024
          }
        }
      }
    ]
  },
  
  plugins: [
    new CleanWebpackPlugin(),
    
    new HtmlWebpackPlugin({
      template: './public/index.html',
      minify: !isDevelopment
    }),
    
    new MiniCssExtractPlugin({
      filename: isDevelopment
        ? '[name].css'
        : '[name].[contenthash:8].css',
      chunkFilename: isDevelopment
        ? '[id].css'
        : '[id].[contenthash:8].css'
    }),
    
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
    })
  ],
  
  optimization: {
    minimize: !isDevelopment,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true
          }
        }
      }),
      new CssMinimizerPlugin()
    ],
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        }
      }
    },
    runtimeChunk: 'single'
  },
  
  devServer: {
    static: './dist',
    port: 3000,
    hot: true,
    historyApiFallback: true
  },
  
  devtool: isDevelopment ? 'eval-source-map' : 'source-map'
};
```

## Common Mistakes

1. **Not cleaning dist folder** - use `clean: true` or CleanWebpackPlugin
2. **Forgetting contenthash** - breaks browser caching
3. **Not splitting chunks** - large initial bundles
4. **Including dev dependencies in production** - check externals
5. **Not using source maps** - difficult debugging
6. **Missing optimization** - slow production builds
7. **Incorrect publicPath** - broken assets in deployment
8. **Not configuring caching** - slower rebuild times

## Best Practices

1. Use separate configs for dev and production
2. Enable tree shaking with `sideEffects: false` in package.json
3. Implement proper code splitting strategy
4. Use contenthash for long-term caching
5. Minimize and compress assets in production
6. Use webpack-bundle-analyzer to optimize bundle size
7. Configure proper source maps for each environment
8. Leverage caching for faster rebuilds
9. Use modern JavaScript target for smaller bundles
10. Monitor and set performance budgets

## Key Takeaways

1. Webpack bundles modules and optimizes assets
2. Loaders transform files, plugins extend functionality
3. Code splitting reduces initial bundle size
4. Production builds need minification and optimization
5. Dev server enables hot module replacement
6. Source maps critical for debugging
7. Proper caching strategy improves performance
8. Bundle analysis helps identify optimization opportunities
9. Module federation enables micro-frontends
10. Configuration complexity worth the performance gains

## Resources

- [Webpack Documentation](https://webpack.js.org/)
- [Webpack Bundle Analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [Awesome Webpack](https://github.com/webpack-contrib/awesome-webpack)
- [Webpack Guides](https://webpack.js.org/guides/)
- [SurviveJS Webpack](https://survivejs.com/webpack/)
