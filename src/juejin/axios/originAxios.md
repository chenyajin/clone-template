```ts
import axios from 'axios'
import qs from 'qs'

import type {
  AxiosRequestConfig,
  AxiosInstance,
  AxiosResponse,
  AxiosError,
} from 'axios'
import type { CreateAxiosOptions } from './axiosTransform'
import type { RequestOptions, Result } from '#/axios'

import { isFunction } from '@/utils/is'
import { ElMessage } from 'element-plus'
export * from './axiosTransform'
import { useLoginStoreWithOut } from '@/stores/modules/user/login'
import { getCanvasFingerprint } from '@/utils'
import { getRefreshToken } from '@/utils/auth'

type RequestQueueItem<T = unknown> = {
  instance: AxiosInstance
  conf: CreateAxiosOptions
  options: RequestOptions
  resolve: (data: Promise<T>) => void
  reject: (data: unknown) => void
}
/**
 * axios 模块
 */
export default class OriginAxios {
  private axiosInstance: AxiosInstance
  private readonly options: CreateAxiosOptions
  private isRefreshing = false // 是否正在刷新token，开启请求队列
  private requestQueue: RequestQueueItem[] // 待请求队列

  constructor(options: CreateAxiosOptions) {
    this.options = options
    this.axiosInstance = axios.create(options)
    this.isRefreshing = false
    this.requestQueue = []
  }

  // 获取 transform
  private getTransform() {
    const { transform } = this.options

    return transform
  }

  interceptors(instance: AxiosInstance): void {
    const { requestInterceptors } = this.getTransform() || {}
    // 请求拦截
    instance.interceptors.request.use(
      (config) => {
        if (requestInterceptors && isFunction(requestInterceptors)) {
          config = requestInterceptors(config, this.options)
        }
        return config
      },
      (error) => {
        return Promise.reject(error)
      }
    )
    // 响应拦截
    instance.interceptors.response.use(
      (res: AxiosResponse<any>) => {
        return res
      },
      (error: AxiosError) => {
        throw error
      }
    )
  }

  // 请求管道处理具体细节
  requestPipeline<T = unknown>(
    instance: AxiosInstance,
    conf: CreateAxiosOptions,
    options: RequestOptions,
    resolve: (data: Promise<T>) => void,
    reject: (data: unknown) => void
  ): void {
    const { transform } = this.options
    const { responseInterceptorsCatch, transformResponseHook } = transform || {}

    this.interceptors(instance)

    instance<any, AxiosResponse<Result>>(conf)
      .then((res: AxiosResponse<Result>) => {
        if (transformResponseHook && isFunction(transformResponseHook)) {
          try {
            const ret = transformResponseHook(res, options)
            resolve(ret)
          } catch (err) {
            reject(err || new Error('request error!'))
          }
          return
        }
        resolve(res.data as unknown as Promise<T>)
      })
      .catch((error) => {
        const code = error?.response?.status
        if (code === 401) {
          if (!this.isRefreshing) {
            this.isRefreshing = true
            this.refreshToken()
          } else {
            this.addRequestQueueForRefreshToken<T>(
              instance,
              conf,
              options,
              resolve,
              reject
            )
          }
        } else {
          if (
            responseInterceptorsCatch &&
            isFunction(responseInterceptorsCatch)
          ) {
            reject(responseInterceptorsCatch(error))
            return
          }
        }
      })
  }

  // 刷新token
  refreshToken() {
    const loginStore = useLoginStoreWithOut()
    this.postRefreshTokenFunc()
      .then((res) => {
        if (res.data) {
          const data = res.data
          loginStore.updateToken(data.data)
          this.requestQueueStartAfterRefreshToken()
        }
      })
      .catch(() => {
        loginStore.logout()
        ElMessage.error('登录已失效，需要重新登录')
      })
      .finally(() => {
        this.isRefreshing = false
      })
  }
  postRefreshTokenFunc() {
    const params = {
      clientId: getCanvasFingerprint(),
    }
    const token = getRefreshToken()
    return axios.post(
      import.meta.env.VITE__APP_BASE_URL + '/auth/refresh',
      params,
      {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }
    )
  }

  // 添加请求到等待队列
  addRequestQueueForRefreshToken<T = unknown>(
    instance: AxiosInstance,
    conf: CreateAxiosOptions,
    options: RequestOptions,
    resolve: (data: Promise<T>) => void,
    reject: (data: unknown) => void
  ): void {
    this.requestQueue.push({
      instance,
      conf,
      options,
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      resolve,
      reject,
    })
  }

  // 刷新token成功等待队列开始请求
  requestQueueStartAfterRefreshToken(): void {
    let requestQueueItem = this.requestQueue.pop()

    while (requestQueueItem) {
      const { options, instance, conf, resolve, reject } = requestQueueItem

      this.requestPipeline(instance, conf, options, resolve, reject)

      requestQueueItem = this.requestQueue.pop()
    }
  }

  // 支持 form-data 处理
  supportFormData(config: AxiosRequestConfig) {
    const headers = config.headers || this.options.headers
    const contentType = headers?.['Content-Type'] || headers?.['content-type']

    if (
      contentType !== 'application/x-www-form-urlencoded;charset=UTF-8' ||
      !Reflect.has(config, 'data') ||
      config.method?.toUpperCase() === 'GET'
    ) {
      return config
    }

    return {
      ...config,
      data: qs.stringify(config.data, { arrayFormat: 'brackets' }),
    }
  }

  request<T = any>(
    config: AxiosRequestConfig,
    options?: RequestOptions
  ): Promise<T> {
    let conf: CreateAxiosOptions = config

    // cancelToken 如果被深拷贝，会导致最外层无法使用cancel方法来取消请求
    if (config.cancelToken) {
      conf.cancelToken = config.cancelToken
    }

    const transform = this.getTransform()
    const { requestOptions } = this.options
    // 将默认配置和用户配置组合
    const opt: RequestOptions = Object.assign({}, requestOptions, options)
    const { beforeRequestHook } = transform || {}

    // 请求前的钩子函数
    if (beforeRequestHook && isFunction(beforeRequestHook)) {
      conf = beforeRequestHook(conf, opt)
    }
    conf = this.supportFormData(conf)
    return new Promise((resolve, reject) => {
      this.requestPipeline(this.axiosInstance, conf, opt, resolve, reject)
    })
  }
}
```
