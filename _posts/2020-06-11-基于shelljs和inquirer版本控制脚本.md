---
layout: post
title: 实现一个版本控制脚本
subtitle: 基于shelljs和inquirer版本控制脚本
date: 2020-06-11
author: ZJ
header-img: img/post-bg-debug.png
catalog: true
tags:
  - npm version
  - shelljs
  - inquirer
---

## npm version

话不多说，直接上代码：

```
npm version [<newversion> | major | minor | patch | premajor | preminor | prepatch | prerelease [--preid=<prerelease-id>] | from-git]
```

这是啥意思呢，又有什么用呢？

### 这是啥？

前后端项目中，基本都会有个 package.json，用于管理依赖和项目的一些基本信息，其中有一项：`version` 参数，用来标明项目当前的版本号。`npm` 提供了 `npm version` 命令工具，采用[语义化版本](https://semver.org/lang/zh-CN/)规范。

### 有什么用？

运行 `npm version` 命令，会自动更改项目 package.json 中`version`字段的值，默认会为这次修改做一次`git commit`提交(你也可以通过参数设置不提交)，`npm version`后跟各参数的含义这里不再赘述，可以自行谷歌。

### 怎么用

<blockquote>这里，我通过 shelljs 和 inquirer 写了一个命令脚本，这样你就无需去记那么多的命令参数了，仅供参考：</blockquote>

```
const shelljs = require('shelljs');
const inquirer = require('inquirer');

// 判定git命令是否可用
if (!shelljs.which('git')) {
    // 向命令行打印git命令不可用的提示信息
    shelljs.echo('Sorry, this script requires git');
    // 退出当前进程
    shelljs.exit(1);
}

let userInputVersionType = '';
const masterVersionChoices = [
    { value: 'major', name: '主版本号' },
    { value: 'minor', name: '次版本号' },
    { value: 'patch', name: '补丁号' }
];
const testVersionChoices = [
    { value: 'premajor', name: '预备主版本' },
    { value: 'preminor', name: '预备次版本' },
    { value: 'prepatch', name: '预备补丁' },
    { value: 'prerelease', name: '预发布版本' }
];
const generatorWhiteSpace = needLength => {
    const spaces = [];
    // eslint-disable-next-line no-plusplus
    for (let i = 0; i < needLength; i++) {
        spaces.push(' ');
    }
    return spaces.join('');
};
const generateVersionLine = versionType => {
    const versionChoices =
        versionType === 'official' ? masterVersionChoices : testVersionChoices;
    versionChoices.forEach(item => {
        // eslint-disable-next-line no-param-reassign
        item.name = `${item.value}${generatorWhiteSpace(
            30 - item.value.length - item.name.length
        )}${item.name}`;
    });
    return versionChoices;
};

inquirer
    .prompt([
        {
            type: 'list',
            name: 'versionType',
            message: '请选择修订版本类型：',
            default: true,
            choices: [
                { name: '正式版本', value: 'official' },
                { name: '测试版本', value: 'preview' }
            ]
        }
    ])
    .then(answers => {
        const versionType = answers.versionType;
        return inquirer
            .prompt([
                {
                    type: 'list',
                    name: 'inputVersion',
                    message: '请选择修订版本: ',
                    default: true,
                    choices: generateVersionLine(versionType)
                }
            ])
            .then(answers => {
                userInputVersionType = answers.inputVersion;

                if (versionType === 'preview') {
                    return inquirer
                        .prompt([
                            {
                                type: 'list',
                                name: 'preid',
                                message: '请选择预发布类型',
                                default: true,
                                choices: [
                                    { name: 'alpha内测版本', value: 'alpha' },
                                    { name: 'beta公测版本', value: 'beta' },
                                    { name: 'rc预览版', value: 'rc' }
                                ]
                            }
                        ])
                        .then(preidAnswers => {
                            return {
                                inputVersion: userInputVersionType,
                                preid: preidAnswers.preid
                            };
                        });
                } else {
                    return {
                        inputVersion: userInputVersionType
                    };
                }
            })
            .then(verisonOptions => {
                return Promise.all([
                    verisonOptions,
                    inquirer.prompt([
                        {
                            type: 'input',
                            name: 'inputDesc',
                            message: 'commit msg: '
                        }
                    ])
                ]);
            })
            .then(([versionOptions, commitmsg]) => {
                const { inputVersion, preid } = versionOptions;
                const { inputDesc } = commitmsg;
                let hasPreid = '';
                if (preid) {
                    hasPreid = ` --preid ${preid}`;
                }
                const makeTag = shelljs.exec(
                    `npm version ${inputVersion}${hasPreid} -m 'Upgrade to v%s${
                        inputDesc ? ` for ${inputDesc}` : ''
                    }'`
                );
                if (makeTag.code !== 0) {
                    shelljs.exit();
                }
                shelljs.exec('git push -u origin --tags && git push');

                shelljs.exit();
            });
    })
    .catch(err => {
        console.error(err);
        shelljs.exit();
    });

```
