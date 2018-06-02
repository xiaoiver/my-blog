https://www.youtube.com/watch?v=geKCzi7ZPkA
https://github.com/smooth-code/svgr
https://github.com/facebook/create-react-app/pull/3718
https://github.com/facebook/create-react-app/issues/1388
https://github.com/FWeinb/babel-plugin-named-asset-import

plugins: [
[
    require.resolve('babel-plugin-named-asset-import'),
    {
    loaderMap: {
        svg: {
            ReactComponent: 'svgr/webpack![path]',
        },
    },
    },
],
],
