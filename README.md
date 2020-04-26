# qinkun-analyze

1、如何注册加载子应用

```js
export function registerMicroApps(
  apps: Array<RegistrableApp<T>>,
  lifeCycles?: LifeCycles<T>,
  opts?: RegisterMicroAppsOpts,
) {
  window.__POWERED_BY_QIANKUN__ = true;

  // Each app only needs to be registered once
  const unregisteredApps = apps.filter(app => !microApps.some(registeredApp => registeredApp.name === app.name));

  microApps = [...microApps, ...unregisteredApps];

  let prevAppUnmountedDeferred: Deferred<void>;

  unregisteredApps.forEach(app => {
    const { name, entry, render, activeRule, props = {} } = app;

    registerApplication(
      name,

      async ({ name: appName }) => {
        await frameworkStartedDefer.promise;

        const { getTemplate = identity, ...settings } = opts || {};
        // get the entry html content and script executor
        const { template: appContent, execScripts, assetPublicPath } = await importEntry(entry, {
          // compose the config getTemplate function with default wrapper
          getTemplate: flow(getTemplate, getDefaultTplWrapper(appName)),
          ...settings,
        });

        // as single-spa load and bootstrap new app parallel with other apps unmounting
        // (see https://github.com/CanopyTax/single-spa/blob/master/src/navigation/reroute.js#L74)
        // we need wait to load the app until all apps are finishing unmount in singular mode
        if (await validateSingularMode(singularMode, app)) {
          await (prevAppUnmountedDeferred && prevAppUnmountedDeferred.promise);
        }
        // 第一次加载设置应用可见区域 dom 结构
        // 确保每次应用加载前容器 dom 结构已经设置完毕
        render({ appContent, loading: true });

        let jsSandbox: Window = window;
        let mountSandbox = () => Promise.resolve();
        let unmountSandbox = () => Promise.resolve();
        if (useJsSandbox) {
          const { fetch } = settings || {};
          const sandbox = genSandbox(appName, { fetch });
          jsSandbox = sandbox.sandbox;
          mountSandbox = sandbox.mount;
          unmountSandbox = sandbox.unmount;
        }

        const {
          beforeUnmount = [],
          afterUnmount = [],
          afterMount = [],
          beforeMount = [],
          beforeLoad = [],
        } = mergeWith({}, getAddOns(jsSandbox, assetPublicPath), lifeCycles, (v1, v2) => concat(v1 ?? [], v2 ?? []));

        await execHooksChain(toArray(beforeLoad), app);

        // get the lifecycle hooks from module exports
        let { bootstrap: bootstrapApp, mount, unmount } = await execScripts(jsSandbox);

        if (!isFunction(bootstrapApp) || !isFunction(mount) || !isFunction(unmount)) {
          if (process.env.NODE_ENV === 'development') {
            console.warn(
              `LifeCycles are not found from ${appName} entry exports, fallback to get them from window['${appName}'] `,
            );
          }

          // fallback to global variable who named with ${appName} while module exports not found
          const globalVariableExports = (window as any)[appName] || {};
          bootstrapApp = globalVariableExports.bootstrap;
          // eslint-disable-next-line prefer-destructuring
          mount = globalVariableExports.mount;
          // eslint-disable-next-line prefer-destructuring
          unmount = globalVariableExports.unmount;
          if (!isFunction(bootstrapApp) || !isFunction(mount) || !isFunction(unmount)) {
            throw new Error(`You need to export the functional lifecycles in ${appName} entry`);
          }
        }

        return {
          bootstrap: [bootstrapApp],
          mount: [
            async () => {
              if ((await validateSingularMode(singularMode, app)) && prevAppUnmountedDeferred) {
                return prevAppUnmountedDeferred.promise;
              }

              return undefined;
            },
            // 添加 mount hook, 确保每次应用加载前容器 dom 结构已经设置完毕
            async () => render({ appContent, loading: true }),
            // exec the chain after rendering to keep the behavior with beforeLoad
            async () => execHooksChain(toArray(beforeMount), app),
            mountSandbox,
            mount,
            // 应用 mount 完成后结束 loading
            async () => render({ appContent, loading: false }),
            async () => execHooksChain(toArray(afterMount), app),
            // initialize the unmount defer after app mounted and resolve the defer after it unmounted
            async () => {
              if (await validateSingularMode(singularMode, app)) {
                prevAppUnmountedDeferred = new Deferred<void>();
              }
            },
          ],
          unmount: [
            async () => execHooksChain(toArray(beforeUnmount), app),
            unmount,
            unmountSandbox,
            async () => execHooksChain(toArray(afterUnmount), app),
            // remove the app content after unmount
            async () => render({ appContent: '', loading: false }),
            async () => {
              if ((await validateSingularMode(singularMode, app)) && prevAppUnmountedDeferred) {
                prevAppUnmountedDeferred.resolve();
              }
            },
          ],
        };
      },

      activeRule,
      props,
    );
  });
}
```


2、为什么调用start
