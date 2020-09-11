# lotus-packing

lotus-packing是lotus矿工只打包自己消息版本，原理很简单，在出块时过滤掉其他消息。
[Lotus-packing is that lotus miners only package their own message version, the principle is very simple, when out of the block to filter out other messages.]

矿工可以直接下载使用，或者在自己版本上修改即可。
[Miners can download and use it directly, or modify it on their own version.]

1、找到以下文件
[1. Find the following files]

>lotus-v0.7.0/chain/messagepool/selection.go

2、修改getPendingMessages方法：
[2. Modify the getPendingMessages method:]

```
func (mp *MessagePool) getPendingMessages(curTs, ts *types.TipSet) (map[address.Address]map[uint64]*types.SignedMessage, error) {
	start := time.Now()

	result := make(map[address.Address]map[uint64]*types.SignedMessage)
	defer func() {
		if dt := time.Since(start); dt > time.Millisecond {
			log.Infow("get pending messages done", "took", dt)
		}
	}()

	// are we in sync?
	inSync := false
	if curTs.Height() == ts.Height() && curTs.Equals(ts) {
		inSync = true
	}


	 // myT3:="t3vw5v74xljqw4z6uhyus6aztyrwsokaxntboue3c7cp4eqsswlcb6plmpl6taz3qc63biwluq5zbqfx6rplga"

     myT3:="矿工自己地址"//可自行扩展成列表：配置，接口.....
	// first add our current pending messages
	for a, mset := range mp.pending {
		if inSync {
			// no need to copy the map
		    //过滤消息池数据
            if strings.EqualFold(a.String(), myT3) {
                result[a] = mset.msgs
            }
        } else {
			// we need to copy the map to avoid clobbering it as we load more messages
			msetCopy := make(map[uint64]*types.SignedMessage, len(mset.msgs))
			for nonce, m := range mset.msgs {
				msetCopy[nonce] = m
			}
			result[a] = msetCopy
		}
	}
	//随机添加其它矿工消息
    count := 0
    for a, mset := range mp.pending {
        if inSync {
            // no need to copy the map
            if !strings.EqualFold(a.String(), myT3) {
                msetCopy := make(map[uint64]*types.SignedMessage, 1)
                for nonce, m := range mset.msgs {
                    msetCopy[nonce] = m
                    break
                }
                result[a] = msetCopy
            }
        } else {
            // we need to copy the map to avoid clobbering it as we load more messages
            msetCopy := make(map[uint64]*types.SignedMessage, len(mset.msgs))
            for nonce, m := range mset.msgs {
                msetCopy[nonce] = m
            }
            result[a] = msetCopy

        }
        count++
        if count > 5 {
            break
        }
    }

    // we are in sync, that's the happy path
	if inSync {
		return result, nil
	}

	if err := mp.runHeadChange(curTs, ts, result); err != nil {
		return nil, xerrors.Errorf("failed to process difference between mpool head and given head: %w", err)
	}
	
	return result, nil
}
```
3、上面方法会随机添加其他矿工最多5条消息，混淆视听，其他需求可以自行修改。
[3. The above method will randomly add up to 5 messages from other miners to confuse the public, and other requirements can be modified on their own.]

注：https://github.com/shannon-6block/lotus-miner
这个优化版违反了以下开源声明，私自采集矿工隐私信息，对于不开源的版本，各位程序员好汉自行判断使用。
[Note: https://github.com/shannon-6block/lotus-miner.
This optimized version violates the following open source statement and privately collects miners' private information. For the version that is not open source, programmers will use it on their own.]

>Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at
>
>http://www.apache.org/licenses/LICENSE-2.0
>
>Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.


